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
    onDNSConfigChanged = make(chan string, 1)          // Network change (buffered)
    onLog              = make(chan string, 10)          // Log messages (buffered)
    onShareFileHelper  = make(chan ShareFileHelper, 1) // File helper registration
)
```

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
    Stopping(7)           // Optimistic UI state during teardown
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
      | (no auth)     |    | (multi-user bug)  |
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
      | Stopping  |  (optimistic UI state) -----------------+
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

Used to:
- Show custom server hostname in UI
- Hide Mullvad exit node info for non-Tailscale servers
- Determine feature availability per server type

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

### 5.7 TLS / Certificate Handling

- All TLS validation is handled by Go's `net/http` with system certificate store
- **No** custom certificate pinning or import UI
- Self-signed Headscale servers require the CA to be trusted at the Android system level
- No special timeout or retry logic for unreachable custom servers (standard 30s LocalAPI timeout)

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

**Package Filtering in newBuilder():**
1. MDM `includedPackages` -> only route listed apps (allowlist)
2. MDM `excludedPackages` -> exclude listed apps (blocklist)
3. User-selected packages from SharedPreferences
4. Built-in disallowed: RCS, Android Auto, GoPro, Sonos, Chromecast, voicemail, Google Scone

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

### 7.2 WiFi/Cellular Switching

1. `onLinkPropertiesChanged` fires with new network properties
2. `pickDefaultNetwork()` recalculates best network (prefers WiFi)
3. `Libtailscale.onDNSConfigChanged(interfaceName)` notifies Go
4. Go side: `netmon.InjectEvent()` triggers DNS reconfiguration
5. **No VPN restart needed** - multiTUN handles transition seamlessly

### 7.3 Airplane Mode

1. `onLost()` removes all networks
2. `pickDefaultNetwork()` returns null
3. DNS update skipped (no interface to report)
4. Backend continues running, WireGuard maintains state
5. On airplane mode off: `onAvailable` fires, normal reconnection

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

### 7.7 Multi-User Android Bug

```go
if strings.Contains(err.Error(), "INTERACT_ACROSS_USERS") {
    vpnService.service.UpdateVpnStatus(false)
    return errMultipleUsers
}
```

On some Android devices with multiple user profiles, VPN establishment fails with a security exception. The app gracefully degrades by disabling VPN.

### 7.8 ChromeOS Specifics

- `avoidEmptyDNS` flag set when `IsChromeOS()` returns true
- Falls back to Google DNS servers (`8.8.8.8`, `8.8.4.4`) when no DNS configured
- Old TUNs must be explicitly closed before creating new ones (ChromeOS doesn't auto-close)

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

### 8.2 Custom Server Errors

| Error | Trigger | User Message |
|-------|---------|-------------|
| `INVALID_CUSTOM_URL` | URL validation failure | "Please enter a valid URL in the form https://server.com" |
| `ADD_PROFILE_FAILED` | Backend login failure | "Failed to add profile" |
| Network timeout | Unreachable server | Generic error via ADD_PROFILE_FAILED |
| TLS error | Certificate issues | Generic error via ADD_PROFILE_FAILED |

### 8.3 Health Warning System

**File:** `android/src/main/java/com/tailscale/ipn/ui/health/HealthNotifier.kt`

- **Debounce:** 3-second delay to avoid notification spam
- **State filter:** Only shows warnings when `state == Running`
- **Dependency tracking:** Some warnings suppress related ones
- **Severity levels:** low, medium, high (high triggers system notification)
- **Filtered warnings:** `is-using-unstable-version`, `wantrunning-false`

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

### 8.5 State Persistence for Recovery

| Key | Storage | Purpose |
|-----|---------|---------|
| `privatelogid` | EncryptedSharedPrefs | Analytics log ID |
| `loginmethod` | EncryptedSharedPrefs | OAuth/token login method |
| `customloginserver` | EncryptedSharedPrefs | Custom control URL |
| `isAbleToStartVPN` | SharedPreferences | Service restart eligibility |
| `statestore-*` | EncryptedSharedPrefs | All IPN state (base64) |

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
