# Libtailscale Integration Report: State Machine & Custom Server Connection

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Go-Side Interface (libtailscale)](#2-go-side-interface-libtailscale)
3. [Kotlin/Java Bridge Layer](#3-kotlinjava-bridge-layer)
4. [IPN State Machine](#4-ipn-state-machine)
5. [Custom Control Server Connection Flow](#5-custom-control-server-connection-flow)
6. [VPN Service Lifecycle & TUN Management](#6-vpn-service-lifecycle--tun-management)
7. [Network Edge Cases](#7-network-edge-cases)
8. [Error Handling & Recovery](#8-error-handling--recovery)
9. [File Reference Index](#9-file-reference-index)

---

## 1. Architecture Overview

The Tailscale Android app uses a layered architecture:

```
+---------------------------+
|   Compose UI / ViewModels |  Kotlin (Jetpack Compose)
+---------------------------+
|   Notifier / Client       |  Kotlin (StateFlow, LocalAPI calls)
+---------------------------+
|   JNI Bridge (gomobile)   |  libtailscale.* interfaces
+---------------------------+
|   Go Backend              |  ipnlocal.LocalBackend, wgengine, netstack
+---------------------------+
|   WireGuard / TUN         |  Kernel TUN device via Android VpnService
+---------------------------+
```

Communication between layers:
- **Kotlin -> Go**: Direct method calls via gomobile-generated bindings (`Libtailscale.start()`, `Libtailscale.requestVPN()`, `app.callLocalAPI()`, etc.)
- **Go -> Kotlin**: Callback interfaces (`NotificationCallback.OnNotify()`, `IPNService.Protect()`, `AppContext.Log()`, etc.)
- **Async coordination**: Go channels (`onVPNRequested`, `onDisconnect`, `onDNSConfigChanged`, `onLog`, `onShareFileHelper`)

---

## 2. Go-Side Interface (libtailscale)

### 2.1 Entry Point

```go
// libtailscale/interfaces.go
func Start(dataDir, directFileRoot string, hwAttestationPref bool, appCtx AppContext) Application
```

This is the single entry point called from `App.kt`. It:
1. Initializes logging (redirects stdout/stderr to Android logcat)
2. Sets XDG environment variables (`XDG_CACHE_HOME`, `XDG_CONFIG_HOME`, `HOME`)
3. Creates the `App` struct and spawns `runBackend()` in a goroutine
4. Returns an `Application` handle used for all subsequent calls

### 2.2 Exported Functions (Java -> Go)

| Function | Signature | Purpose |
|----------|-----------|---------|
| `Start` | `(dataDir, directFileRoot, hwAttestPref, appCtx) Application` | Initialize backend |
| `RequestVPN` | `(service IPNService)` | Request VPN tunnel creation |
| `ServiceDisconnect` | `(service IPNService)` | Signal VPN teardown |
| `SendLog` | `(logstr []byte)` | Forward Android log to Go |
| `SetShareFileHelper` | `(helper ShareFileHelper)` | Register Taildrop file handler |
| `OnDNSConfigChanged` | `(ifname string)` | Notify network/DNS change |
| `CallLocalAPI` | `(timeout, method, endpoint, body) LocalAPIResponse` | HTTP-like API call to backend |
| `CallLocalAPIMultipart` | `(timeout, method, endpoint, parts) LocalAPIResponse` | Multipart file upload |
| `EditPrefs` | `(prefs MaskedPrefs) LocalAPIResponse` | Shortcut to PATCH `/prefs` |
| `WatchNotifications` | `(mask, callback) NotificationManager` | Subscribe to IPN bus |
| `NotifyPolicyChanged` | `()` | Trigger MDM policy reload |

### 2.3 Callback Interfaces (Go -> Java)

**AppContext** - Android application context:
```
Log, EncryptToPref, DecryptFromPref, GetStateStoreKeysJSON,
GetOSVersion, GetDeviceName, GetInstallSource,
ShouldUseGoogleDNSFallback, IsChromeOS,
GetInterfacesAsJson, GetPlatformDNSConfig,
GetSyspolicyStringValue, GetSyspolicyBooleanValue,
GetSyspolicyStringArrayJSONValue,
HardwareAttestationKey{Supported,Create,Release,Public,Sign,Load}
```

**IPNService** - VPN service handle:
```
ID, Protect, NewBuilder, Close, DisconnectVPN, UpdateVpnStatus
```

**VPNServiceBuilder** - TUN configuration:
```
SetMTU, AddDNSServer, AddSearchDomain, AddRoute,
ExcludeRoute, AddAddress, Establish
```

**NotificationCallback** - IPN bus events:
```
OnNotify(jsonBytes []byte) error
```

**ShareFileHelper** - Taildrop file operations:
```
OpenFileWriter, GetFileURI, RenameFile, ListFilesJSON,
OpenFileReader, DeleteFile, GetFileInfo
```

### 2.4 Global Channels (Async Communication)

```go
// libtailscale/callbacks.go
var (
    onVPNRequested     = make(chan IPNService)         // VPN start signal
    onDisconnect       = make(chan IPNService)         // VPN stop signal
    onGoogleToken      = make(chan string)             // OAuth token
    onDNSConfigChanged = make(chan string, 1)          // Network change (buffered=1)
    onLog              = make(chan string, 10)          // Log messages (buffered=10)
    onShareFileHelper  = make(chan ShareFileHelper, 1) // File helper registration
)
```

#### Edge Cases: Channel Semantics

| Channel | Buffer | Edge Case |
|---------|--------|-----------|
| `onVPNRequested` | 0 (unbuffered) | `RequestVPN()` blocks until `runBackend()` receives. If backend is stuck in `updateTUN()`, the Kotlin caller hangs on the JNI thread. |
| `onDisconnect` | 0 (unbuffered) | `ServiceDisconnect()` blocks until consumed. If Go backend is processing a slow `updateTUN()`, the Android service's `close()` path hangs. |
| `onDNSConfigChanged` | 1 | Non-blocking send with `default` clause. If buffer is full, the DNS change is **silently dropped**. Rapid WiFi/cellular toggling can lose intermediate DNS configs. |
| `onLog` | 10 | If log buffer fills (e.g., during high-throughput logging), `SendLog()` blocks the Kotlin logging thread. |
| `onShareFileHelper` | 1 | Non-blocking send. If called before backend consumes the previous helper, the new helper is dropped. |

### 2.5 Core Go Structs

**App** (application singleton):
```go
type App struct {
    dataDir, directFileRoot string
    shareFileHelper         ShareFileHelper
    appCtx                  AppContext
    store                   *stateStore        // Encrypted state persistence
    policyStore             *syspolicyStore    // MDM policy reader
    logIDPublicAtomic       atomic.Pointer[logid.PublicID]
    localAPIHandler         http.Handler       // LocalAPI HTTP handler
    backend                 *ipnlocal.LocalBackend
    ready                   sync.WaitGroup     // 2 signals: init + backend start
    backendMu               sync.Mutex
}
```

**backend** (internal engine state):
```go
type backend struct {
    engine       wgengine.Engine
    backend      *ipnlocal.LocalBackend
    sys          *tsd.System
    devices      *multiTUN              // Hot-swappable TUN multiplexer
    settings     settingsFunc
    lastCfg      *router.Config
    lastDNSCfg   *dns.OSConfig
    netMon       *netmon.Monitor
    logIDPublic  logid.PublicID
    logger       *logtail.Logger
    bus          *eventbus.Bus
    avoidEmptyDNS bool                  // ChromeOS Google DNS fallback
    appCtx       AppContext
}
```

**VpnService** (global singleton):
```go
type VpnService struct {
    service    IPNService   // Current Android VPN service handle
    fd         int32        // TUN file descriptor
    fdDetached bool         // Whether FD ownership transferred
}
var vpnService = &VpnService{}
```

#### Edge Case: VpnService Global Singleton Has No Synchronization

`vpnService` is a package-level global with **no mutex**. It is written from:
- `runBackend()` via `onVPNRequested` case (sets `.service`)
- `runBackend()` via `onDisconnect` case (sets `.service = nil`)
- `updateTUN()` (sets `.fd`, `.fdDetached`)

And read from:
- `runBackend()` main loop (checks `vpnService.service != nil`)
- `updateTUN()` (reads `.service`)

Since all writes happen in the same goroutine (`runBackend`), this is safe **only because Go's event loop is single-threaded**. If any future refactoring adds concurrent access, this becomes a data race.

### 2.6 Backend Initialization Sequence

```
Start()
  -> initLogging()           // Pipe stdout/stderr to logcat
  -> newApp()
       -> newStateStore()    // EncryptedSharedPreferences via AppContext
       -> syspolicyStore{}   // MDM policy store
       -> RegisterInterfaceGetter()   // Network interface provider
       -> RegisterStore()    // Policy store registration
       -> RegisterHardwareAttestationKeyFns()  // If hardware attestation enabled
       -> go watchFileOpsChanges()
       -> go runBackend()
            -> hostinfo.SetOSVersion/Package/DeviceModel
            -> newBackend()
                 -> tsd.NewSystem()
                 -> netmon.New()           // Network monitor
                 -> setupLogs()            // Logtail remote logging
                 -> VPNFacade{}            // Router + DNS configurator
                 -> wgengine.NewUserspaceEngine()
                 -> netstack.Create()      // Userspace networking
                 -> ipnlocal.NewLocalBackend()
                 -> taildrop.SetFileOps()
                 -> lb.Start(ipn.Options{})
            -> localapi.NewHandler()
            -> app.ready.Done() x2        // Signal initialization complete
            -> [MAIN EVENT LOOP]          // select{} on channels
```

#### Edge Cases: Initialization

1. **`app.ready` WaitGroup with count=2**: `CallLocalAPI()` calls `a.ready.Wait()` before proceeding. If `runBackend()` panics before calling `Done()` twice, all LocalAPI callers block forever. The Kotlin side has a timeout per request, but the **Notifier's `watchNotifications()` has no such timeout** — it would hang indefinitely.

2. **`newBackend()` panic**: If `wgengine.NewUserspaceEngine()` or `ipnlocal.NewLocalBackend()` panics, the `defer app.ready.Done()` won't fire (panics don't execute defers on other goroutines). The app enters a permanent "loading" state with no visible error.

3. **Multiple `Start()` calls**: Nothing prevents Kotlin from calling `Libtailscale.start()` twice (e.g., if `startLibtailscale()` is reentered during SAF directory change). This would create duplicate `App` instances and duplicate `runBackend()` goroutines competing for the same channels.

4. **`stateStore` encryption failure**: `newStateStore()` uses `EncryptedSharedPreferences` via the `AppContext` interface. If the Android Keystore is corrupted (happens after OS upgrades), `DecryptFromPref()` fails silently, returning empty strings. The backend starts with no persisted state, effectively resetting the user's session.

---

## 3. Kotlin/Java Bridge Layer

### 3.1 Application Class (App.kt)

`App` extends `UninitializedApp` and implements `libtailscale.AppContext`.

**Initialization chain:**
```
Application.onCreate()
  -> Register MDM receiver
  -> Create notification channels (STATUS, FILE, HEALTH)
  -> initOnce() [thread-safe singleton]
       -> initializeApp()
            -> Restore Taildrop directory URI
            -> startLibtailscale(directFileRoot, hardwareAttestation)
                 -> Libtailscale.start(filesDir, directFileRoot, hwAttest, this)
                 -> ShareFileHelper.init()
                 -> Request.setApp(app)
                 -> Notifier.setApp(app)
                 -> Notifier.start(applicationScope)
            -> NetworkChangeCallback.monitorDnsChanges()
            -> Create ViewModels
            -> Collect state, prefs, MDM settings
```

#### Edge Cases: App Initialization

1. **`appInstance` is `lateinit` but NOT volatile** (App.kt): If `App.get()` is called from a background thread before `onCreate()` completes, it crashes with `UninitializedPropertyAccessException`. The `initOnce()` function uses `@Volatile initialized` flag but `appInstance` itself is not volatile — there's a narrow window where `initialized` is `true` but `appInstance` is not yet visible to other threads.

2. **`startLibtailscale()` called multiple times**: When the SAF Taildrop directory URI changes, `startLibtailscale()` is called again with a new `directFileRoot`. This overwrites the `app` field without cleaning up the previous Go backend instance. Creates duplicate `Notifier` watchers and native resource leaks.

3. **`onTerminate()` never guaranteed**: Android documentation states `onTerminate()` is "never called on production devices." The cleanup of `Notifier.stop()`, `notificationManager.cancelAll()`, `applicationScope.cancel()`, `viewModelStore.clear()`, and `unregisterReceiver(mdmChangeReceiver)` may never execute.

4. **`getInterfacesAsJson()` null/exception handling**: `NetworkInterface.getNetworkInterfaces()` can return null on some OEM ROMs. `Collections.list(null)` throws NPE. The code wraps individual interface processing in try-catch but the outer `getNetworkInterfaces()` call is not guarded.

5. **EncryptedSharedPreferences corruption**: `encryptToPref()`/`decryptFromPref()` wrap Android Keystore operations. On some devices after OTA updates, the master key becomes invalid. The code catches exceptions and returns empty strings, causing silent state loss.

### 3.2 Notifier (Notifier.kt)

Central hub for all IPN bus notifications. Publishes `StateFlow` values:

| StateFlow | Type | Description |
|-----------|------|-------------|
| `state` | `Ipn.State` | Backend connection state |
| `netmap` | `Netmap.NetworkMap?` | Peer topology |
| `prefs` | `Ipn.Prefs?` | User/backend preferences |
| `engineStatus` | `Ipn.EngineStatus?` | WireGuard engine stats |
| `browseToURL` | `String?` | Login URL to open in browser |
| `loginFinished` | `String?` | Login completion marker |
| `health` | `Health.State?` | Health warnings |
| `outgoingFiles` | `List<OutgoingFile>?` | Taildrop outgoing |
| `incomingFiles` | `List<PartialFile>?` | Taildrop incoming |
| `filesWaiting` | `Empty.Message?` | Files ready for pickup |

**Watch mask:**
```kotlin
Netmap(8) | Prefs(4) | InitialState(2) | InitialHealthState(128) | RateLimitNetmaps(256)
```

**Processing:**
```kotlin
app.watchNotifications(mask) { notification ->
    val notify = decoder.decodeFromStream<Notify>(notification.inputStream())
    notify.State?.let { state.set(Ipn.State.fromInt(it)) }
    notify.NetMap?.let(netmap::set)
    notify.Prefs?.let(prefs::set)
    notify.BrowseToURL?.let(browseToURL::set)
    notify.LoginFinished?.let { loginFinished.set(it.property) }
    notify.Health?.let(health::set)
    // ...
}
```

#### Edge Cases: Notifier

1. **JSON deserialization not protected**: `decoder.decodeFromStream<Notify>()` can throw `SerializationException` if the Go backend sends a notification with a new/unknown field or an incompatible format. There's no try-catch wrapper. A single malformed notification **kills the entire notification pipeline** — the app loses all future state updates.

2. **StateFlow drops intermediate states**: `StateFlow` only retains the latest value. If the backend fires `NoState → Starting → Running` rapidly, a slow subscriber might only see `Running`, never seeing `Starting`. UI elements that show a "Connecting..." spinner based on `Starting` state may never display.

3. **`browseToURL` one-shot semantics**: `browseToURL` is a `StateFlow<String?>`. Once set, it persists as the current value. If the user rotates the device, the `collectAsState()` in `MainActivity` re-collects the same URL and opens the browser again. The code attempts to clear it by checking `loginFinished`, but there's a race window.

4. **Notification callback blocks Go goroutine**: The `OnNotify()` callback is called synchronously from Go. If Kotlin-side processing (JSON parsing + StateFlow updates) is slow, it blocks the Go notification goroutine, which can back-pressure the entire IPN bus.

### 3.3 LocalAPI Client (Client.kt)

All control operations go through the LocalAPI, which is an HTTP-like interface served by `ipnlocal.LocalBackend`:

| Operation | Method | Endpoint |
|-----------|--------|----------|
| Edit prefs | PATCH | `/localapi/v0/prefs` |
| Start backend | POST | `/localapi/v0/start` |
| Login interactive | POST | `/localapi/v0/login-interactive` |
| Switch profile | POST | `/localapi/v0/profiles/{id}` |
| Get status | GET | `/localapi/v0/status` |
| File put | PUT | `/localapi/v0/file-put/{stableID}` |

#### Edge Cases: LocalAPI Client

1. **Response handler called twice in error path** (Client.kt): In the request execution flow, on certain failure paths the result callback `responseHandler()` can fire twice — once with the failure result that falls through (not returned), and again in the catch block. This causes duplicate state updates.

2. **No concurrency limits**: Nothing prevents the UI from firing 100+ simultaneous LocalAPI calls (e.g., rapid tapping). Each call creates a new `httpx.Request` and the Go side processes them sequentially on a single `localAPIHandler`, but the Kotlin coroutines pile up waiting.

3. **Multipart file upload has 24-hour timeout**: File uploads use a hardcoded 24-hour timeout with no per-request override. A flaky network can leave uploads hanging for hours.

4. **InputStreams not closed on partial failure**: If `openInputStream()` fails for one file part in a multipart upload, earlier opened streams may leak if the exception is thrown before the cleanup `finally` block.

5. **`app.ready.Wait()` blocks on Go thread**: `CallLocalAPI()` calls `a.ready.Wait()` which blocks until Go backend initialization completes. If called from the main thread (via a misconfigured coroutine dispatcher), this causes an ANR.

---

## 4. IPN State Machine

### 4.1 State Definitions

```kotlin
enum class State(val value: Int) {
    NoState(0),           // Initial / unknown
    InUseOtherUser(1),    // VPN owned by different Android user
    NeedsLogin(2),        // Authentication required
    NeedsMachineAuth(3),  // Admin approval required
    Stopped(4),           // Authenticated, VPN off
    Starting(5),          // VPN initializing
    Running(6),           // VPN active
    Stopping(7)           // Optimistic UI state during teardown (Kotlin-only)
}
```

### 4.2 State Transition Diagram

```
                    +-----------+
                    |  NoState  |  (app just launched)
                    +-----+-----+
                          |
              +-----------+-----------+
              |                       |
              v                       v
      +-------+-------+    +---------+---------+
      |  NeedsLogin   |    | InUseOtherUser(1) |
      | (no auth)     |    | (multi-user)      |
      +-------+-------+    +-------------------+
              |
              | User authenticates (BrowseToURL flow)
              v
      +-------+-----------+
      | NeedsMachineAuth  |  [optional - admin must approve device]
      +-------+-----------+
              |
              | Admin approves
              v
      +-------+-------+
      |    Stopped    |  (authenticated, VPN off)
      +--+----+----+--+
         |    |    |
         |    |    +-------- setWantRunning(false) <--------+
         |    |                                              |
         |    +--- restart after OOM (if isAbleToStartVPN)   |
         |                                                   |
         | setWantRunning(true) + VPN permission granted     |
         v                                                   |
      +--+--------+                                          |
      | Starting  |  (TUN being configured)                  |
      +--+--------+                                          |
         |                                                   |
         | updateTUN() succeeds                              |
         v                                                   |
      +--+--------+                                          |
      |  Running  |  (VPN active, traffic flowing)           |
      +--+--------+                                          |
         |                                                   |
         | User stops / permission revoked / error           |
         v                                                   |
      +--+--------+                                          |
      | Stopping  |  (optimistic Kotlin-side state) --------+
      +-----------+
```

### 4.3 Key State Predicates

```kotlin
val ableToStartVPN = state > Ipn.State.NeedsMachineAuth  // i.e., Stopped/Starting/Running
val vpnRunning = state == Ipn.State.Starting || state == Ipn.State.Running
```

### 4.4 Go-Side State Machine (runBackend main loop)

The Go backend runs an event loop in `runBackend()` that selects on:

```go
select {
case s := <-stateCh:
    // IPN state changed (NeedsLogin, Running, etc.)
    // If state >= Starting && VPN service available && config changed -> updateTUN()

case nm := <-netmapCh:
    // Network map changed (peers added/removed)

case cfg := <-configs:
    // VPNFacade received new router/DNS config from engine
    // Store config, signal configErrs channel

case s := <-onVPNRequested:
    // Android VPN service ready
    // Set socket protect function, rebind magicsock
    // If state >= Starting -> updateTUN()

case s := <-onDisconnect:
    // VPN service disconnected
    // Close TUNs, clear protect function

case i := <-onDNSConfigChanged:
    // Network interface changed
    // Inject netmon event for DNS reconfig
}
```

### 4.5 State Machine Edge Cases

#### 4.5.1 Stopping State is Kotlin-Only (Optimistic)

The `Stopping(7)` state is set **only** by Kotlin code in `IPNService.close()`:
```kotlin
Notifier.setState(Ipn.State.Stopping)
```

The Go backend **never** produces state value 7. This creates a divergence: Kotlin shows "Stopping" while Go is still in `Running(6)`. If the disconnect fails or is slow, the UI shows "Stopping" indefinitely with no timeout to recover.

#### 4.5.2 State Regression Without Guard

Nothing prevents the Go backend from sending state transitions that appear to "regress" (e.g., `Running → NeedsLogin` when auth key expires). The Kotlin side applies these directly:
```kotlin
notify.State?.let { state.set(Ipn.State.fromInt(it)) }
```

If this happens while the VPN service is still running, the UI shows "NeedsLogin" but the VPN tunnel may still be partially active, confusing users.

#### 4.5.3 `InUseOtherUser` Is a Terminal State

When state = `InUseOtherUser(1)`, the VPN is owned by another Android user profile. There is no automatic recovery. The user must:
1. Switch to the other Android profile
2. Stop Tailscale there
3. Return to their profile

The app shows an error but provides no guidance on resolution.

#### 4.5.4 Race Between `stateCh` and `onVPNRequested`

In the `select{}` loop:
```go
case s := <-onVPNRequested:
    // Sets socket protect, calls updateTUN() if state >= Starting
case s := <-stateCh:
    // Calls updateTUN() if state >= Starting && service != nil
```

If `stateCh` delivers `Starting` **before** `onVPNRequested` delivers the service handle, the `stateCh` case sees `vpnService.service == nil` and skips `updateTUN()`. Later when `onVPNRequested` fires, it checks state but state may have already advanced. The TUN is established correctly due to config comparison (`lastCfg != cfg`), but there's a brief window where the backend is in `Starting` state with no TUN.

#### 4.5.5 Config Channel Ordering

```go
case cfg := <-configs:
    b.lastCfg = cfg.rcfg
    b.lastDNSCfg = cfg.dcfg
    configErrs <- nil
```

The `VPNFacade` sends configs through the `configs` channel and blocks on `configErrs`. If the main loop is stuck processing a slow `updateTUN()` on the `stateCh` case, the `VPNFacade` goroutine (from the WireGuard engine) blocks, which can back-pressure the entire WireGuard engine.

#### 4.5.6 Profile Switch During Active Connection

During `switchProfile()`:
```kotlin
Client.editPrefs(MaskedPrefs { WantRunning = false })  // Step 1: Stop
Client.switchProfile(profile)                           // Step 2: Switch
startVPN()                                              // Step 3: Start new
```

Each step is an independent LocalAPI call. If the app is killed between step 1 and step 3, the VPN is stopped but the profile is either not switched (killed between 1-2) or switched but not started (killed between 2-3). On restart, the service uses `isAbleToStartVPN` which may be stale from the previous profile.

---

## 5. Custom Control Server Connection Flow

### 5.1 UI Entry Point

**File:** `android/src/main/java/com/tailscale/ipn/ui/view/CustomLogin.kt`

Users access custom server login via:
`User Switcher -> Menu -> "Use an alternate server"`

The UI presents:
- Title: "Use an alternate server"
- Description mentioning Headscale compatibility
- Text field with placeholder `https://my.custom.server.com`
- "Add account" submit button

### 5.2 URL Validation

**File:** `android/src/main/java/com/tailscale/ipn/ui/viewModel/CustomLoginViewModel.kt`

```kotlin
fun setControlURL(urlStr: String, onSuccess: () -> Unit) {
    val valid = urlStr.startsWith("http", ignoreCase = true)
              && urlStr.contains("://")
              && urlStr.length > 7
    if (!valid) {
        errorDialog.set(ErrorDialogType.INVALID_CUSTOM_URL)
        return
    }
    loginWithCustomControlURL(urlStr) { result ->
        result.onFailure { errorDialog.set(ErrorDialogType.ADD_PROFILE_FAILED) }
        result.onSuccess { onSuccess() }
    }
}
```

Validation rules:
- Must start with `http` (case-insensitive)
- Must contain `://`
- Must be > 7 characters
- No hostname resolution or TLS probe performed client-side

#### Edge Cases: URL Validation

1. **Accepts invalid URLs**: `httpx://evil.com` passes validation (starts with `http`, contains `://`, > 7 chars). `http://` alone (length 7) is rejected but `http://x` (length 8) passes despite being likely invalid.

2. **No trailing slash normalization**: `https://headscale.example.com` and `https://headscale.example.com/` may be treated differently by the Go backend's control client.

3. **No port validation**: `https://server:99999` passes Kotlin validation but will fail at the TCP level in Go.

4. **Unicode/IDN hostnames**: Punycode URLs like `https://xn--example.com` pass but may fail depending on Go's DNS resolver handling of IDN.

5. **Whitespace not trimmed**: Leading/trailing spaces in the URL would pass validation but fail connection.

### 5.3 Complete Connection Sequence

```
User enters URL (e.g., "https://headscale.example.com")
    |
    v
[1] CustomLoginViewModel.setControlURL(url)
    |  - Validates URL format
    v
[2] IpnViewModel.loginWithCustomControlURL(url)
    |  - Creates MaskedPrefs { ControlURL = url }
    v
[3] IpnViewModel.login(maskedPrefs)
    |
    +--[3a] Client.editPrefs(maskedPrefs)
    |       |  PATCH /localapi/v0/prefs
    |       |  Sets ControlURL + WantRunning=true in Go backend
    |       v
    +--[3b] Client.start(Ipn.Options{UpdatePrefs})
    |       |  POST /localapi/v0/start
    |       |  LocalBackend begins connecting to custom control server
    |       v
    +--[3c] Client.startLoginInteractive()
            |  POST /localapi/v0/login-interactive
            |  Backend generates auth URL on custom server
            v
[4] Go backend contacts custom control server
    |  - Sends device registration request
    |  - Receives login URL
    |  - Broadcasts Notify{BrowseToURL: "https://headscale.example.com/..."}
    v
[5] Notifier.browseToURL fires on Kotlin side
    |  - MainActivity opens Chrome Custom Tabs with login URL
    v
[6] User authenticates in browser
    |  - Custom server validates credentials
    |  - Issues node key
    v
[7] Go backend receives auth completion
    |  - Broadcasts Notify{LoginFinished}
    |  - State transitions: NeedsLogin -> NeedsMachineAuth (optional) -> Starting -> Running
    v
[8] VPN tunnel established
    |  - onVPNRequested channel fires
    |  - updateTUN() configures Android VPN
    |  - State: Running
    v
[9] UI navigates home, shows connected status
```

#### Edge Cases: Connection Sequence

1. **Steps 3a-3c are sequential but not atomic**: Each is an independent LocalAPI call. If the app crashes after `editPrefs` sets the ControlURL but before `startLoginInteractive`, on restart the backend has the custom URL but won't automatically prompt for login. The user sees a stale "NeedsLogin" state with no browser prompt.

2. **`editPrefs` succeeds but `start` fails**: The ControlURL is persisted but the backend isn't started. The profile now points to the custom server but is in a limbo state. There's no rollback mechanism.

3. **`BrowseToURL` arrives before UI is ready**: If the Go backend responds extremely fast, `browseToURL` StateFlow is set before `MainActivity` starts collecting it. Since `StateFlow` retains the latest value, it will be collected when the activity starts — but if the activity recreates (configuration change), it re-opens the browser.

4. **User cancels browser authentication**: The app shows `NeedsLogin` state indefinitely. There's no timeout. The custom control server profile exists but is unauthenticated. The user must manually delete the profile or re-initiate login.

5. **Custom server returns non-standard auth URL**: If the server returns a `BrowseToURL` that's not a valid HTTP URL (e.g., a custom scheme), `Intent(Intent.ACTION_VIEW, Uri.parse(url))` may fail to find a handler, crashing or silently failing.

6. **Network change during authentication**: If the user switches WiFi networks between steps 4-7, the Go backend may lose connectivity to the custom server. The auth flow stalls with no retry. The user sees "NeedsLogin" but the browser tab shows a completed login — out of sync.

### 5.4 Custom Server Detection

```kotlin
// IpnLocal.LoginProfile
fun isUsingCustomControlServer(): Boolean {
    return ControlURL != null && ControlURL != "https://controlplane.tailscale.com"
}

fun customControlServerHostname(): String? {
    if (!isUsingCustomControlServer()) return null
    return try { URL(ControlURL).host } catch (e: Exception) { null }
}
```

#### Edge Cases: Detection

1. **Default URL comparison is fragile**: The check compares against the literal string `"https://controlplane.tailscale.com"`. If the Go backend normalizes it (e.g., adds trailing slash, lowercases), the comparison fails and a Tailscale-server user appears to be on a "custom" server.

2. **`URL(ControlURL).host` on non-standard URLs**: If `ControlURL` is `http://192.168.1.1:8080/path`, `host` returns `192.168.1.1` which is correct. But for IPv6 like `http://[::1]:8080`, the brackets in `host` depend on the URL parser implementation.

3. **Feature gating based on server type**: Mullvad exit nodes, auto-updates, and other features are hidden for custom servers. If a Headscale server supports these features, users have no way to enable them.

### 5.5 Server Switching (Profile-Based)

```kotlin
fun switchProfile(profile: IpnLocal.LoginProfile, completionHandler: ...) {
    // 1. Stop current connection
    Client.editPrefs(MaskedPrefs { WantRunning = false })

    // 2. Switch to new profile
    Client.switchProfile(profile)  // POST /profiles/{id}

    // 3. Restart VPN with new profile's ControlURL
    startVPN()
}
```

State transitions during switch:
```
Running -> [editPrefs WantRunning=false] -> Stopped
        -> [switchProfile] -> NeedsLogin (new profile loaded)
        -> [startVPN] -> Starting -> Running
```

#### Edge Cases: Profile Switching

1. **Non-atomic multi-step operation**: Three independent API calls with no transaction guarantees. Kill between steps leaves inconsistent state.

2. **Switching to expired profile**: If the target profile's auth key expired while another profile was active, `switchProfile` succeeds but `startVPN` results in `NeedsLogin`. The UI doesn't distinguish "expired" from "never authenticated."

3. **Rapid profile switches**: User taps multiple profiles quickly. Each fires the 3-step sequence. Steps from different switches can interleave: `stop(A) -> switch(B) -> switch(C) -> start(B's VPN)` — the wrong profile's VPN starts.

4. **Profile deleted server-side**: If the custom server deletes the device while another profile is active, switching back to that profile results in an auth error with no clear message.

### 5.6 MDM Forced Custom Server

**File:** `android/src/main/res/xml/app_restrictions.xml`
```xml
<restriction android:key="LoginURL"
    android:restrictionType="string"
    android:title="Custom control server URL" />
```

MDM administrators can force a control server URL via Android's `RestrictionsManager`. This is read by:
1. `App.getSyspolicyStringValue("LoginURL")` on Kotlin side
2. `syspolicyStore.ReadString()` on Go side
3. Backend applies it as the `ControlURL` preference

#### Edge Cases: MDM

1. **MDM URL overrides user profiles**: If a user has multiple profiles with different servers, the MDM `LoginURL` overrides all of them. Switching profiles still changes the displayed server name in the UI, but the actual connection goes to the MDM-enforced URL. Confusing and misleading.

2. **MDM policy change while connected**: If MDM pushes a new `LoginURL` while the VPN is active, `NotifyPolicyChanged()` is called but the existing connection is **not** torn down. The old server remains active until the next reconnect.

3. **MDM `LoginURL` not validated**: The MDM policy value goes directly to the Go backend without the Kotlin URL validation. A malformed MDM URL causes a backend error with no user-facing message.

### 5.7 TLS / Certificate Handling

- All TLS validation is handled by Go's `net/http` with system certificate store
- **No** custom certificate pinning or import UI
- Self-signed Headscale servers require the CA to be trusted at the Android system level
- No special timeout or retry logic for unreachable custom servers (standard 30s LocalAPI timeout)

#### Edge Cases: TLS

1. **No certificate error feedback**: When TLS fails against a custom server (self-signed, expired, wrong hostname), the error propagates as a generic `ADD_PROFILE_FAILED`. Users see no indication that it's a certificate issue.

2. **Android user CA restriction**: On Android 7+, user-installed CAs are not trusted by default for apps targeting API 24+. Users must configure the device in a special way or the admin must provision the CA via MDM.

3. **Certificate expiry during active session**: If the custom server's TLS certificate expires while connected, existing WireGuard tunnels continue (WireGuard uses its own crypto), but control plane communication fails. Health warnings may fire but the connection appears "Working" to the user.

---

## 6. VPN Service Lifecycle & TUN Management

### 6.1 IPNService (Android VpnService)

**File:** `android/src/main/java/com/tailscale/ipn/IPNService.kt`

```kotlin
class IPNService : VpnService(), libtailscale.IPNService
```

**Intent Actions:**

| Action | Behavior | Return |
|--------|----------|--------|
| `ACTION_START_VPN` | Show notification, setWantRunning(true), requestVPN | START_STICKY |
| `ACTION_STOP_VPN` | setWantRunning(false), close() | START_NOT_STICKY |
| `ACTION_RESTART_VPN` | Stop then start | START_NOT_STICKY |
| `android.net.VpnService` | Always-On VPN trigger | START_STICKY |
| null (system restart) | If isAbleToStartVPN, restart | START_STICKY or NOT_STICKY |

#### Edge Cases: Service Lifecycle

1. **CoroutineScope never cancelled**: `IPNService` creates `val scope = CoroutineScope(Dispatchers.IO)` but never calls `scope.cancel()` in `onDestroy()`. Long-running coroutines (like `showForegroundNotification`) survive service destruction, leaking resources.

2. **START_VPN / STOP_VPN rapid succession**: `showForegroundNotification()` is launched async via `scope.launch`. If STOP_VPN arrives before the notification is posted, `startForeground()` is called after `stopSelf()`, causing `IllegalStateException` on Android 12+.

3. **`close()` called from multiple paths without reentrancy guard**: `close()` is called from `onStartCommand(STOP)`, `onDestroy()`, and `onRevoke()`. It calls `Notifier.setState(Stopping)`, `disconnectVPN()` (which calls `stopSelf()`), and `Libtailscale.serviceDisconnect()`. If `onRevoke()` fires concurrently with `onDestroy()`, `serviceDisconnect()` is called twice with the same service handle.

4. **`onRevoke()` while `onStartCommand()` executing**: `onRevoke()` can fire from a system thread at any time. If it fires while `onStartCommand(START_VPN)` is setting up the VPN, both paths try to manipulate the same service state concurrently with no synchronization.

5. **Foreground service start restriction (Android 12+)**: `showForegroundNotification()` calls `startForeground()`. On Android 12+, this throws `ForegroundServiceStartNotAllowedException` if the service wasn't started from an allowed context. The code catches `IllegalStateException` in `startVPN()` but **not** inside `showForegroundNotification()` itself.

6. **`DisconnectVPN()` called from Go thread**: The `libtailscale.IPNService` interface method `DisconnectVPN()` is called from Go's `runBackend()`. It calls `stopSelf()` which must happen on the service's main thread. gomobile marshals this correctly, but if the service is already destroyed, `stopSelf()` is a no-op — however the Go side doesn't know this.

**Package Filtering in newBuilder():**
1. MDM `includedPackages` -> only route listed apps (allowlist)
2. MDM `excludedPackages` -> exclude listed apps (blocklist)
3. User-selected packages from SharedPreferences
4. Built-in disallowed: RCS, Android Auto, GoPro, Sonos, Chromecast, voicemail, Google Scone

#### Edge Cases: Package Filtering

1. **Uninstalled package names**: `addDisallowedApplication()` throws `PackageManager.NameNotFoundException` if the package isn't installed. The built-in disallowed list includes packages like GoPro and Sonos that most users don't have — these throw silently (caught by the wrapping try-catch in the Go bridge).

2. **Conflicting allowlist/blocklist**: If MDM sets both `includedPackages` and `excludedPackages`, the code uses `includedPackages` only (allowlist wins). But user-selected excluded packages are ignored in this case with no UI indication.

3. **Self-exclusion**: If the Tailscale app is added to the excluded packages list (accidentally or via MDM), the app loses connectivity to the control server through the VPN, potentially causing a disconnect loop.

### 6.2 Multi-TUN Architecture

**File:** `libtailscale/multitun.go`

Android VPN interfaces have static configurations, but WireGuard expects a single mutable TUN. The `multiTUN` solves this by multiplexing:

```go
type multiTUN struct {
    devices  chan tun.Device   // Queue for adding new devices
    events   chan tun.Event    // Combined events
    reads    chan ioRequest    // Read from oldest device
    writes   chan ioRequest    // Write to newest device
}
```

**Lifecycle:**
1. New TUN created -> sent to `devices` channel
2. Previous TUN drained and closed
3. Reads from oldest (draining) device
4. Writes to newest (active) device
5. When oldest reaches EOF, removed from queue
6. Seamless transition with no packet loss

#### Edge Cases: multiTUN

1. **Rapid TUN replacements**: If `updateTUN()` is called many times rapidly (e.g., during rapid DNS reconfiguration), devices queue up in the channel. Each old device must be drained before being closed. If packets keep arriving on old devices, draining takes longer, increasing memory usage.

2. **`Shutdown()` vs `Close()` semantics**: `Shutdown()` sends to `d.shutdowns` and waits on `d.shutdownDone`. `Close()` closes `d.close` and waits on `d.closeErr`. If both are called (e.g., `CloseTUNs()` then the WireGuard engine closes the device), the behavior depends on which channel is read first in the `select{}` — potential double-close.

3. **Read/write routing during transition**: During the brief window between adding a new device and the old device reaching EOF, reads come from the old device and writes go to the new device. If the old device's config differs (different routes), packets read from it may be for routes that the new device doesn't have.

4. **EOF detection**: The `multiTUN` relies on reads returning EOF to know a device is done. If the old TUN device doesn't produce EOF (kernel bug, ChromeOS behavior), the draining phase hangs, and the device is never removed from the queue.

### 6.3 TUN Update Flow (updateTUN)

**File:** `libtailscale/net.go`

```go
func (b *backend) updateTUN(rcfg *router.Config, dcfg *dns.OSConfig) error {
    b.CloseTUNs()                    // Close previous TUNs (required for ChromeOS)

    builder := vpnService.service.NewBuilder()
    builder.SetMTU(1280)             // defaultMTU

    // DNS servers (with ChromeOS Google DNS fallback)
    for _, dns := range nameservers { builder.AddDNSServer(dns) }
    for _, dom := range searchDomains { builder.AddSearchDomain(dom) }

    // Routes
    for _, route := range rcfg.Routes { builder.AddRoute(route) }
    for _, route := range rcfg.LocalRoutes { builder.ExcludeRoute(route) }  // API 33+

    // Addresses
    for _, addr := range rcfg.LocalAddrs { builder.AddAddress(addr) }

    // Establish
    parcelFD, err := builder.Establish()
    if err != nil {
        if strings.Contains(err.Error(), "INTERACT_ACROSS_USERS") {
            return errMultipleUsers  // Android multi-user bug
        }
        return err
    }

    if parcelFD == nil { return errVPNNotPrepared }

    tunFD, _ := parcelFD.Detach()    // Transfer FD ownership to Go
    tunDev, _, _ := tun.CreateUnmonitoredTUNFromFD(int(tunFD))
    b.devices.add(tunDev)            // Add to multiTUN

    return nil
}
```

#### Edge Cases: updateTUN

1. **`CloseTUNs()` before new TUN**: There's a gap between closing old TUNs and establishing the new one. During this window, all VPN traffic is **black-holed** — packets are silently dropped. For always-on VPN with "Block connections without VPN" enabled in Android settings, this causes a brief but complete network outage.

2. **`parcelFD.Detach()` error ignored**: The error from `Detach()` is assigned to `_`. If detachment fails (e.g., FD already closed by another thread), `tunFD` is 0 or invalid, and `CreateUnmonitoredTUNFromFD(0)` operates on stdin.

3. **MTU hardcoded to 1280**: The minimum IPv6 MTU. This is conservative but means larger packets require fragmentation, reducing throughput. If the underlying link supports larger MTUs (WiFi typically 1500), performance is unnecessarily degraded.

4. **Empty route list**: If `rcfg.Routes` is empty (e.g., during a config transition), `builder.Establish()` creates a VPN with no routes. Android may accept this but all traffic continues via the default route, not the VPN. When routes are later populated, a new TUN is needed.

5. **`AddRoute` with overlapping CIDRs**: If the route config contains overlapping CIDRs (e.g., `10.0.0.0/8` and `10.1.0.0/16`), Android's VPN builder accepts both but behavior is implementation-defined per OEM.

6. **IPv6 address without IPv6 route**: If `LocalAddrs` contains an IPv6 address but `Routes` has no IPv6 routes, the TUN has an IPv6 address but no traffic is routed to it. This silently breaks IPv6 connectivity within the tailnet.

### 6.4 VPNFacade (Router + DNS Configurator)

**File:** `libtailscale/vpnfacade.go`

Implements both `router.Router` and `dns.OSConfigurator`:

```go
type VPNFacade struct {
    SetBoth           func(rcfg *router.Config, dcfg *dns.OSConfig) error
    GetBaseConfigFunc func() (dns.OSConfig, error)
    mu                sync.Mutex
    rcfg              *router.Config
    dcfg              *dns.OSConfig
}
```

- `Set(rcfg)` and `SetDNS(dcfg)` store configs under mutex
- `ReconfigureVPN()` calls `SetBoth()` which sends configs through channel to main loop
- `SupportsSplitDNS()` returns false (Android limitation)

#### Edge Cases: VPNFacade

1. **`Set()` and `SetDNS()` called from different goroutines**: The mutex protects individual reads/writes, but `Set()` followed by `SetDNS()` from different goroutines can result in `ReconfigureVPN()` being called with a stale `rcfg` or `dcfg` pairing. The `SetBoth` function receives whatever is stored at the time of the call.

2. **`SupportsSplitDNS() = false`**: This means MagicDNS has to route **all** DNS queries through the Tailscale DNS proxy, not just `.ts.net` queries. On slow custom servers, this adds latency to all DNS resolution.

3. **`ReconfigureVPN()` blocks on channel**: If the main event loop is busy, `ReconfigureVPN()` blocks, which blocks the WireGuard engine's goroutine. The engine can't process new handshakes or peer changes while blocked.

### 6.5 Socket Protection

```go
// When VPN service connects:
netns.SetAndroidProtectFunc(func(fd int) error {
    if !s.Protect(int32(fd)) {
        log.Printf("[unexpected] VpnService.protect(%d) returned false", fd)
    }
    return nil  // Best-effort, don't fail on protect errors
})
b.backend.DebugRebind()  // Rebind existing sockets with protection
```

This prevents WireGuard traffic from being routed back through the VPN tunnel (routing loop prevention).

#### Edge Cases: Socket Protection

1. **Protect returns false but error ignored**: `Protect()` returning false means the socket could NOT be excluded from the VPN. All subsequent WireGuard traffic through that socket routes back into the VPN tunnel (routing loop). The code logs this but continues, meaning **silent complete VPN failure**.

2. **Race between `Protect()` and TUN teardown**: If the VPN service is being torn down while a new socket is created, `Protect()` is called on a service that no longer owns the VPN. The behavior is undefined per Android docs.

3. **`DebugRebind()` not idempotent**: Calling `DebugRebind()` causes magicsock to close and re-create all sockets. If called during active WireGuard sessions, all DERP and direct connections are dropped and must be re-established. Brief connectivity blip.

---

## 7. Network Edge Cases

### 7.1 Network Change Detection

**File:** `android/src/main/java/com/tailscale/ipn/NetworkChangeCallback.kt`

Monitors all networks with `INTERNET + NOT_VPN` capabilities:

```kotlin
NetworkRequest.Builder()
    .addCapability(NET_CAPABILITY_INTERNET)
    .addCapability(NET_CAPABILITY_NOT_VPN)  // Exclude VPN networks (loop prevention)
    .build()
```

**Callbacks handled:**
- `onAvailable` - New network registered
- `onCapabilitiesChanged` - Metered/unmetered status changed
- `onLinkPropertiesChanged` - DNS servers or domains changed
- `onLost` - Network lost

**Default network selection prefers non-metered (WiFi > Cellular).**

#### Edge Cases: Network Monitoring

1. **ReentrantLock held while calling native code**: The lock in `NetworkChangeCallback` is held while calling `Libtailscale.onDNSConfigChanged()`. If the Go side calls back to Kotlin (e.g., `getInterfacesAsJson()`) on the same thread, and that call needs the lock, **deadlock**. While `ReentrantLock` is reentrant for the same thread, gomobile calls may be dispatched on a different thread.

2. **`onLinkPropertiesChanged` before `onAvailable`**: Android doesn't guarantee callback ordering. If `onLinkPropertiesChanged` fires before `onAvailable`, the network isn't in `activeNetworks` yet. The code creates a new `NetworkInfo` entry in `onAvailable`, but DNS servers from the earlier `onLinkPropertiesChanged` are lost.

3. **DNS change coalescing gap**: The `onDNSConfigChanged` Go channel has buffer=1 with non-blocking send. If two DNS changes arrive within the Go event loop's processing time, the second is dropped. This means after WiFi→cellular→WiFi switching, the final DNS config may not be applied.

4. **No explicit debounce in Kotlin layer**: Each callback immediately computes `pickDefaultNetwork()` and calls `onDNSConfigChanged`. Rapid network flapping (e.g., unstable WiFi) causes cascading reconfigurations.

5. **Captive portal networks**: A network with `NET_CAPABILITY_INTERNET` may be behind a captive portal. The callback adds it to `activeNetworks` and may select it as default, routing DNS through the portal's DNS server which may not resolve tailnet names.

### 7.2 WiFi/Cellular Switching

1. `onLinkPropertiesChanged` fires with new network properties
2. `pickDefaultNetwork()` recalculates best network (prefers WiFi)
3. `Libtailscale.onDNSConfigChanged(interfaceName)` notifies Go
4. Go side: `netmon.InjectEvent()` triggers DNS reconfiguration
5. **No VPN restart needed** - multiTUN handles transition seamlessly

#### Edge Cases: WiFi/Cellular

1. **Dual-stack network mismatch**: If WiFi provides IPv4-only and cellular provides IPv6-only, `pickDefaultNetwork()` picks WiFi (non-metered) but tailnet peers reachable only via IPv6 become unreachable.

2. **WiFi without internet**: A connected WiFi network may lack actual internet connectivity. `NET_CAPABILITY_INTERNET` only means the network *claims* to have internet, not that it actually does. DNS queries through this network fail.

3. **Metered WiFi**: Hotel/airplane WiFi may be metered. `pickDefaultNetwork()` still prefers it over cellular because it checks `NOT_METERED` capability, and falls back to any network. But the metered WiFi may have restrictive firewalls blocking WireGuard UDP.

### 7.3 Airplane Mode

1. `onLost()` removes all networks
2. `pickDefaultNetwork()` returns null
3. DNS update skipped (no interface to report)
4. Backend continues running, WireGuard maintains state
5. On airplane mode off: `onAvailable` fires, normal reconnection

#### Edge Cases: Airplane Mode

1. **WiFi-in-airplane-mode**: Users can enable WiFi while in airplane mode. A single `onAvailable` fires. If the WiFi network is different from before airplane mode, stale DNS config persists until `onLinkPropertiesChanged` fires.

2. **Bluetooth tethering**: Not monitored by the callback (doesn't have `NET_CAPABILITY_INTERNET` in some implementations). VPN may not route through Bluetooth-tethered connection.

### 7.4 VPN Permission Revocation

```kotlin
override fun onRevoke() {
    close()  // Sets Stopping state, calls serviceDisconnect
    updateVpnStatus(false)
}
```

Go side receives `onDisconnect`:
```go
case s := <-onDisconnect:
    b.CloseTUNs()
    netns.SetAndroidProtectFunc(nil)
    vpnService.service = nil
```

#### Edge Cases: Permission Revocation

1. **Another VPN app installed**: Android only allows one VPN at a time. Installing and enabling another VPN app calls `onRevoke()` on Tailscale. The app correctly stops, but the Quick Settings tile may still show "Connected" until the next tile update.

2. **`onRevoke()` during `updateTUN()`**: If revocation happens while Go is in the middle of `updateTUN()`, `builder.Establish()` returns null (permission lost). The error is `errVPNNotPrepared` which triggers `closeVpnService()`, which tries to call `DisconnectVPN()` on a service that's already being revoked.

### 7.5 Always-On VPN

Android triggers with intent action `"android.net.VpnService"`:
```kotlin
"android.net.VpnService" -> {
    app.setWantRunning(true)
    Libtailscale.requestVPN(this)
    START_STICKY
}
```

MDM can also force this via `forceEnabled` policy, which hides the disconnect action in notifications.

#### Edge Cases: Always-On VPN

1. **Always-On with "Block without VPN"**: If the user enables Android's "Block connections without VPN" setting and Tailscale crashes, **all network connectivity is lost** until Tailscale restarts. Combined with `START_STICKY`, the system tries to restart, but if the crash is deterministic (e.g., corrupted state), the device has no network.

2. **Always-On VPN + Profile switch**: While Always-On is enabled, switching profiles stops the VPN briefly. During this window, "Block without VPN" drops all traffic. If the new profile needs authentication (browser login), the browser can't reach the internet.

3. **Boot-time startup**: Always-On VPN starts the service during boot. `App.get()` triggers full initialization. If the device has disk encryption and the credential storage isn't yet available, `EncryptedSharedPreferences` fails, and the backend starts with no saved state.

### 7.6 OOM Kill and Service Restart

```kotlin
else -> {  // null intent = system restart after kill
    if (UninitializedApp.get().isAbleToStartVPN()) {
        App.get()  // Re-initialize app
        Libtailscale.requestVPN(this)
        START_STICKY
    } else {
        START_NOT_STICKY  // Don't restart
    }
}
```

`isAbleToStartVPN` is persisted in SharedPreferences and set to true when `state > NeedsMachineAuth`.

#### Edge Cases: OOM Recovery

1. **Stale `isAbleToStartVPN`**: If the user logged out (state = NeedsLogin) but the SharedPreference wasn't updated (race, crash), the service restarts and tries to start a VPN that can't authenticate. The backend enters NeedsLogin but the VPN service is running, showing misleading notifications.

2. **Go backend state lost**: OOM kill destroys the entire process including Go memory. On restart, the Go backend re-reads from `stateStore` (EncryptedSharedPreferences). If the last state wasn't flushed, the backend starts with stale peer keys and must re-negotiate.

3. **Rapid OOM kill loop**: If the device is under extreme memory pressure, the service starts, allocates memory for Go runtime, gets killed again. `START_STICKY` causes another restart. This loop consumes battery and CPU. Android's exponential backoff for service restarts mitigates this but doesn't prevent it entirely.

### 7.7 Multi-User Android Bug

```go
if strings.Contains(err.Error(), "INTERACT_ACROSS_USERS") {
    vpnService.service.UpdateVpnStatus(false)
    return errMultipleUsers
}
```

On some Android devices with multiple user profiles, VPN establishment fails with a security exception. The app gracefully degrades by disabling VPN.

#### Edge Cases: Multi-User

1. **Work profile**: Android Enterprise work profiles create a separate user space. If Tailscale is installed in both personal and work profiles, only one can own the VPN. The loser gets `InUseOtherUser` state with no automatic resolution.

2. **Error detection is string-based**: `strings.Contains(err.Error(), "INTERACT_ACROSS_USERS")` is fragile. If Android changes the error message text in a future version, this check fails and the error is treated as a generic VPN failure.

### 7.8 ChromeOS Specifics

- `avoidEmptyDNS` flag set when `IsChromeOS()` returns true
- Falls back to Google DNS servers (`8.8.8.8`, `8.8.4.4`) when no DNS configured
- Old TUNs must be explicitly closed before creating new ones (ChromeOS doesn't auto-close)

#### Edge Cases: ChromeOS

1. **Google DNS fallback leaks queries**: When `avoidEmptyDNS` triggers the Google DNS fallback, DNS queries for tailnet domains (e.g., `myhost.ts.net`) are sent to Google's resolvers, which can't resolve them. These queries fail with NXDOMAIN.

2. **ChromeOS detection**: `IsChromeOS()` checks `Build.DEVICE` name. New ChromeOS devices or custom builds may not match the known device names, silently disabling the ChromeOS workarounds.

---

## 8. Error Handling & Recovery

### 8.1 VPN Establishment Failure

```go
func (a *App) closeVpnService(err error, b *backend) {
    log.Printf("VPN update failed: %v", err)

    // Disable VPN via preferences
    mp := new(ipn.MaskedPrefs)
    mp.WantRunning = false
    mp.WantRunningSet = true
    a.EditPrefs(*mp)

    // Cleanup
    b.lastCfg = nil
    b.CloseTUNs()
    vpnService.service.DisconnectVPN()
    vpnService.service = nil
}
```

#### Edge Cases: VPN Failure Recovery

1. **`EditPrefs` failure in closeVpnService**: If `EditPrefs` fails (e.g., LocalAPI handler not ready), `WantRunning` stays `true`. On next state transition, the backend tries to establish the VPN again, hitting the same error, creating an infinite retry loop.

2. **`DisconnectVPN()` called with nil service**: If `closeVpnService` is called after `vpnService.service` is already nil (double-close), this panics with nil pointer dereference.

3. **VPN failure notification gap**: After `closeVpnService`, the state transitions to `Stopped`. There's no user-facing notification explaining *why* the VPN stopped. The user sees the VPN turn off with no error message.

### 8.2 Custom Server Errors

| Error | Trigger | User Message |
|-------|---------|-------------|
| `INVALID_CUSTOM_URL` | URL validation failure | "Please enter a valid URL in the form https://server.com" |
| `ADD_PROFILE_FAILED` | Backend login failure | "Failed to add profile" |
| Network timeout | Unreachable server | Generic error via ADD_PROFILE_FAILED |
| TLS error | Certificate issues | Generic error via ADD_PROFILE_FAILED |

#### Edge Cases: Error Reporting

1. **All backend errors map to one message**: `ADD_PROFILE_FAILED` covers DNS resolution failure, TLS errors, HTTP 403, server timeout, invalid response format, and more. Users can't distinguish between "server is down" and "URL is wrong."

2. **No retry mechanism**: After `ADD_PROFILE_FAILED`, the user must manually re-enter the URL and try again. There's no "retry" button.

3. **Partial state left after failure**: If `editPrefs(ControlURL=custom)` succeeds but the login fails, the profile exists with the custom URL. Subsequent attempts to "add" the same server may conflict with the existing profile.

### 8.3 Health Warning System

**File:** `android/src/main/java/com/tailscale/ipn/ui/health/HealthNotifier.kt`

- **Debounce:** 3-second delay to avoid notification spam
- **State filter:** Only shows warnings when `state == Running`
- **Dependency tracking:** Some warnings suppress related ones
- **Severity levels:** low, medium, high (high triggers system notification)
- **Filtered warnings:** `is-using-unstable-version`, `wantrunning-false`

#### Edge Cases: Health Warnings

1. **3-second debounce hides rapid issues**: If a health warning appears and disappears within 3 seconds (e.g., brief DERP disconnection), the user never sees it. If this happens repeatedly, it indicates a problem but is never surfaced.

2. **Warnings suppressed during non-Running states**: Health warnings during `Starting` state are dropped. If the VPN fails to start *because* of a health issue (e.g., DERP region unreachable), the health warning that would explain the failure is suppressed.

3. **`distinctUntilChanged` only checks warning count**: The comparison `old?.Warnings?.count() == new?.Warnings?.count()` means if one warning disappears and a different one appears (count stays same), the UI doesn't update.

### 8.4 Foreground Service Exceptions

```kotlin
try {
    pendingIntent.send()
} catch (e: IllegalStateException) {
    // ForegroundServiceStartNotAllowedException (Android 12+)
    TSLog.e(TAG, "startVPN hit ForegroundServiceStartNotAllowedException: $e")
} catch (e: SecurityException) {
    TSLog.e(TAG, "startVPN hit SecurityException: $e")
}
```

#### Edge Cases: Foreground Service

1. **Exception caught but no recovery**: When `ForegroundServiceStartNotAllowedException` is caught, the VPN simply doesn't start. There's no retry, no user notification, and no state update. The user may think the VPN is starting when it's not.

2. **`showForegroundNotification()` throws before `startForeground()`**: If notification creation throws (e.g., invalid channel ID, notification permission denied on Android 13+), the service never becomes foreground and is killed within 5 seconds by the system.

3. **Notification ID conflicts**: `StartVPNWorker` and `IPNService` both use notification ID 1. The status notification overwrites the worker notification and vice versa, hiding critical user-facing messages.

### 8.5 Taildrop File Transfer Edge Cases

1. **SAF permission revocation**: The user grants a directory via Storage Access Framework. This permission can be revoked at any time (app data clear, storage reset). `ShareFileHelper` calls `openWriterFD()` which throws `SecurityException` with no clear error message to the sending peer.

2. **`runBlocking` on IO dispatcher**: `ShareFileHelper` uses `runBlocking { waitUntilTaildropDirReady() }` which blocks the calling Go goroutine. If the Taildrop directory prompt is never answered by the user, the Go goroutine **blocks forever**, eventually exhausting the goroutine pool.

3. **File deleted during transfer**: `getFileURI()` calls `findFile()` which returns null if the file was deleted between creation and retrieval. IOException thrown but the Go side receives a generic error.

4. **Storage full during write**: `openWriterFD()` creates the file entry but writing fails when storage is full. The partial file remains on disk with no automatic cleanup.

5. **Large file names**: File names with special characters (emojis, path separators, null bytes) may cause issues with SAF's `createFile()` or with the Go backend's file path handling.

### 8.6 Quick Settings Tile Edge Cases

1. **`isRunning` and `currentTile` not atomically updated**: Between `setVPNRunning()` changing `isRunning` and `updateTile()` reading it, `onStopListening()` can set `currentTile = null`, causing NPE in `updateTile()`.

2. **`onTileClick()` TOCTOU**: Checks `needsToStop` under lock, releases lock, then takes action. State can change between check and action. The stop button can start the VPN and vice versa.

3. **Tile state persistence**: Quick Settings tiles are recreated by the system. If the service is killed, the tile shows stale state until `updateTile()` is called.

### 8.7 Worker Edge Cases

1. **`StartVPNWorker` calls `VpnService.prepare()` from background thread**: This is a UI operation that may show a dialog. Called from `WorkManager`'s background thread, it can throw `IllegalStateException` on Android 10+.

2. **No retry policy**: `OneTimeWorkRequest.Builder()` has no retry policy. Transient failures are permanent.

3. **Concurrent worker execution**: Multiple `StartVPNWorker` instances can run concurrently if enqueued rapidly. Each calls `app.startVPN()`, potentially causing multiple service start intents.

### 8.8 State Persistence for Recovery

| Key | Storage | Purpose |
|-----|---------|---------|
| `privatelogid` | EncryptedSharedPrefs | Analytics log ID |
| `loginmethod` | EncryptedSharedPrefs | OAuth/token login method |
| `customloginserver` | EncryptedSharedPrefs | Custom control URL |
| `isAbleToStartVPN` | SharedPreferences | Service restart eligibility |
| `statestore-*` | EncryptedSharedPrefs | All IPN state (base64) |

#### Edge Cases: State Persistence

1. **SharedPreferences cross-process access**: If the VPN service runs in a separate process (not the case currently, but possible with `android:process` attribute), SharedPreferences is not process-safe. Reads may return stale values.

2. **EncryptedSharedPreferences atomic operations**: `EncryptToPref` is not atomic with `DecryptFromPref`. If the app crashes between encrypting a new value and committing it, the preference file may be corrupted.

3. **Base64 encoding of state**: IPN state is base64-encoded before encryption. Double encoding (base64 of encrypted bytes of base64 of state) increases storage size by ~78%. For large netmaps, this can hit SharedPreferences size limits.

---

## 9. File Reference Index

### Go Files (libtailscale/)

| File | Purpose |
|------|---------|
| `interfaces.go` | `Start()`, `Application` interface, `AppContext` interface |
| `backend.go` | `runBackend()` main loop, `newBackend()`, VPN state machine |
| `net.go` | `updateTUN()`, DNS config, route management, VPN error types |
| `multitun.go` | `multiTUN` hot-swap TUN multiplexer |
| `vpnfacade.go` | `VPNFacade` router + DNS configurator |
| `callbacks.go` | Global channels, `RequestVPN`, `ServiceDisconnect`, `OnDNSConfigChanged` |
| `localapi.go` | `CallLocalAPI`, `EditPrefs`, HTTP request/response handling |
| `notifier.go` | `WatchNotifications`, JSON marshaling to Kotlin |
| `store.go` | `stateStore` encrypted persistence |
| `syspolicy_handler.go` | MDM policy store |
| `tailscale.go` | Constants, logging setup |
| `hardware_attestation.go` | Hardware-backed key management |
| `net_interfaces.go` | Network interface enumeration |
| `share_file_ops.go` | Taildrop SAF file operations |

### Kotlin Files (android/src/main/java/com/tailscale/ipn/)

| File | Purpose |
|------|---------|
| `App.kt` | Application singleton, `AppContext` impl, initialization |
| `UninitializedApp.kt` | Base class for pre-init state, VPN start/stop |
| `IPNService.kt` | VPN service lifecycle, `IPNService` impl |
| `VPNServiceBuilder.kt` | TUN configuration builder |
| `NetworkChangeCallback.kt` | Network monitoring, DNS change detection |
| `ShareFileHelper.kt` | Taildrop file operations |
| `TSLog.kt` | Logging bridge |
| `MainActivity.kt` | Navigation, VPN permission, deep links |
| `ui/notifier/Notifier.kt` | IPN bus notification hub |
| `ui/viewModel/IpnViewModel.kt` | Login flow orchestration |
| `ui/viewModel/CustomLoginViewModel.kt` | Custom server URL validation |
| `ui/view/CustomLogin.kt` | Custom server UI |
| `ui/view/UserSwitcherView.kt` | Profile/server switching UI |
| `ui/model/Ipn.kt` | State enum, Prefs, MaskedPrefs models |
| `ui/model/IpnLocal.kt` | LoginProfile, custom server detection |
| `ui/localapi/Client.kt` | LocalAPI HTTP client |
| `ui/health/HealthNotifier.kt` | Health warning management |
| `mdm/MDMSettings.kt` | MDM policy definitions |
| `QuickToggleService.java` | Quick Settings tile |
| `IPNReceiver.java` | Broadcast receiver for VPN intents |
| `StartVPNWorker.java` | WorkManager VPN start task |
| `StopVPNWorker.java` | WorkManager VPN stop task |
| `UseExitNodeWorker.kt` | WorkManager exit node selection |
