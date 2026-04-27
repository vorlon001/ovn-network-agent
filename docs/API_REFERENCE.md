# OVN Network Agent - API Reference

## Table of Contents
1. [Main Package](#main-package)
2. [Agent Package](#agent-package)
3. [OVN Package](#ovn-package)
4. [Routing Package](#routing-package)
5. [OVS Package](#ovs-package)
6. [nftables Package](#nftables-package)
7. [Config Package](#config-package)

---

## Main Package

### Functions

#### `main()`
```go
func main()
```
Entry point for the OVN Network Agent.

**Behavior:**
1. Loads configuration from CLI flags, environment variables, or config file
2. Sets up structured logging
3. Creates and runs the agent
4. Handles SIGINT/SIGTERM signals for graceful shutdown

**Parameters:** None

**Returns:** None

**Example:**
```bash
ovn-network-agent --config /etc/ovn-network-agent/config.yaml
```

#### `loadConfig(args []string) (Config, error)`
```go
func loadConfig(args []string) (Config, error)
```
Loads configuration with priority: CLI flags > environment variables > config file > defaults.

**Parameters:**
- `args`: Command line arguments (typically `os.Args[1:]`)

**Returns:**
- `Config`: Loaded configuration
- `error`: Error if configuration is invalid or version requested

**Example:**
```go
cfg, err := loadConfig(os.Args[1:])
if err != nil {
    if errors.Is(err, errVersionRequested) {
        fmt.Println("ovn-network-agent", version)
        os.Exit(0)
    }
    slog.Error("configuration error", "error", err)
    os.Exit(1)
}
```

#### `setupLogging(level string)`
```go
func setupLogging(level string)
```
Configures structured logging with the specified level.

**Parameters:**
- `level`: Log level ("debug", "info", "warn", "error")

**Returns:** None

**Example:**
```go
setupLogging("debug")
```

---

## Agent Package

### Type: Agent

```go
type Agent struct {
    cfg                Config
    ovn                *OVNClient
    routing            *RouteManager
    reconcileCh        chan struct{}
    effectiveFilters   []*net.IPNet
    consecutiveReAdds  int
    missingChassis     map[string]time.Time
    staleCleanupJitter time.Duration
}
```

Main OVN route synchronization agent.

#### `NewAgent(cfg Config) (*Agent, error)`
```go
func NewAgent(cfg Config) (*Agent, error)
```
Creates a new Agent instance.

**Parameters:**
- `cfg`: Configuration for the agent

**Returns:**
- `*Agent`: New agent instance
- `error`: Error if creation fails

**Example:**
```go
agent, err := NewAgent(cfg)
if err != nil {
    slog.Error("failed to create agent", "error", err)
    os.Exit(1)
}
```

#### `Run(ctx context.Context) error`
```go
func (a *Agent) Run(ctx context.Context) error
```
Starts the agent: connects to OVN, runs initial reconciliation, then loops on events and periodic reconciliation.

**Parameters:**
- `ctx`: Context for cancellation

**Returns:**
- `error`: Error if agent exits with error

**Behavior:**
1. Checks bridge device
2. Ensures bridge IP
3. Enables proxy ARP
4. Sets up veth leak
5. Sets up port forwarding
6. Connects to OVN with retry
7. Restores drained gateways
8. Initial reconciliation
9. Main event loop

**Example:**
```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

if err := agent.Run(ctx); err != nil && ctx.Err() == nil {
    slog.Error("agent exited with error", "error", err)
    os.Exit(1)
}
```

#### `reconcile()`
```go
func (a *Agent) reconcile()
```
Ensures the local routing state matches the desired state from OVN.

**Parameters:** None

**Returns:** None

**Behavior:**
1. Gets OVN state snapshot
2. Computes effective network filters
3. Builds hairpin MAC map
4. Ensures OVN gateway routing
5. Ensures active priority lead
6. Ensures OVS flows
7. Reconciles OVS hairpin flows
8. Reconciles kernel routes
9. Reconciles FRR routes
10. Reconciles veth leak networks
11. Reconciles FRR prefix-list
12. Checks stale chassis

#### `cleanup()`
```go
func (a *Agent) cleanup()
```
Removes all managed resources.

**Parameters:** None

**Returns:** None

**Behavior:**
1. Removes all kernel routes
2. Removes all FRR routes
3. Removes OVS flows
4. Tears down veth leak
5. Tears down port forwarding
6. Removes bridge IP

#### `triggerReconcile()`
```go
func (a *Agent) triggerReconcile()
```
Requests an asynchronous reconciliation (non-blocking).

**Parameters:** None

**Returns:** None

**Behavior:**
- Sends to reconcile channel if not already pending

---

## OVN Package

### Type: OVNClient

```go
type OVNClient struct {
    sbClient      ovsdbClient
    nbClient      ovsdbClient
    state         *OVNState
    cfg           Config
    ctx           context.Context
    onChange      func()
    ready         bool
    debounceMu    sync.Mutex
    stateTimer    *time.Timer
    reconcileTimer *time.Timer
}
```

OVN database client wrapper.

#### `NewOVNClient(cfg Config, onChange func()) *OVNClient`
```go
func NewOVNClient(cfg Config, onChange func()) *OVNClient
```
Creates a new OVNClient instance.

**Parameters:**
- `cfg`: Configuration
- `onChange`: Callback function for state changes

**Returns:**
- `*OVNClient`: New OVN client instance

#### `Connect(ctx context.Context) error`
```go
func (o *OVNClient) Connect(ctx context.Context) error
```
Connects to OVN Southbound and Northbound databases.

**Parameters:**
- `ctx`: Context for cancellation

**Returns:**
- `error`: Error if connection fails

**Behavior:**
1. Gets local hostname
2. Connects to SB DB with retry
3. Monitors SB tables
4. Connects to NB DB with retry
5. Monitors NB tables
6. Marks as ready
7. Performs initial state refresh

#### `Close()`
```go
func (o *OVNClient) Close()
```
Closes OVSDB connections and cancels pending timers.

**Parameters:** None

**Returns:** None

#### `GetState() OVNState`
```go
func (o *OVNClient) GetState() OVNState
```
Returns a snapshot of the current OVN state.

**Parameters:** None

**Returns:**
- `OVNState`: State snapshot

#### `refreshState(ctx context.Context)`
```go
func (o *OVNClient) refreshState(ctx context.Context)
```
Performs a unified state refresh from both OVN databases.

**Parameters:**
- `ctx`: Context for cancellation

**Returns:** None

**Behavior:**
1. Lists SB Port_Binding and Chassis
2. Lists NB Logical_Router_Port, Logical_Router, NAT
3. Builds chassis hostname map
4. Finds local chassisredirect ports
5. Resolves local LRP UUIDs
6. Finds routers with local LRPs
7. Extracts discovered networks
8. Filters NAT entries
9. Extracts SNAT IPs from SB gateway ports
10. Updates state atomically

#### `EnsureGatewayRouting(ctx context.Context, localRouters []LocalRouterInfo, bridgeMAC string) error`
```go
func (o *OVNClient) EnsureGatewayRouting(ctx context.Context, localRouters []LocalRouterInfo, bridgeMAC string) error
```
Ensures that each locally-active router has a default route and static MAC binding.

**Parameters:**
- `ctx`: Context for cancellation
- `localRouters`: List of locally-active routers
- `bridgeMAC`: MAC address of the bridge device

**Returns:**
- `error`: Error if operations fail

#### `EnsureActivePriorityLead(ctx context.Context, localRouters []LocalRouterInfo, localChassisName string) error`
```go
func (o *OVNClient) EnsureActivePriorityLead(ctx context.Context, localRouters []LocalRouterInfo, localChassisName string) error
```
Ensures that for each locally-active router the local Gateway_Chassis entry has a strictly higher priority than all peers.

**Parameters:**
- `ctx`: Context for cancellation
- `localRouters`: List of locally-active routers
- `localChassisName`: Local chassis name

**Returns:**
- `error`: Error if operations fail

#### `DrainGateways(ctx context.Context, localChassisName string) error`
```go
func (o *OVNClient) DrainGateways(ctx context.Context, localChassisName string) error
```
Lowers Gateway_Chassis priority to 0 for graceful shutdown.

**Parameters:**
- `ctx`: Context for cancellation
- `localChassisName`: Local chassis name

**Returns:**
- `error`: Error if operations fail

#### `RestoreDrainedGateways(ctx context.Context, localChassisName string)`
```go
func (o *OVNClient) RestoreDrainedGateways(ctx context.Context, localChassisName string)
```
Restores Gateway_Chassis priority to 1 after restart.

**Parameters:**
- `ctx`: Context for cancellation
- `localChassisName`: Local chassis name

**Returns:** None

#### `CleanupStaleChassisEntries(ctx context.Context, localChassisName string) error`
```go
func (o *OVNClient) CleanupStaleChassisEntries(ctx context.Context, localChassisName string) error
```
Removes OVN NB entries from chassis that have disappeared from the SB Chassis table.

**Parameters:**
- `ctx`: Context for cancellation
- `localChassisName`: Local chassis name

**Returns:**
- `error`: Error if operations fail

### Type: OVNState

```go
type OVNState struct {
    mu                 sync.RWMutex
    LocalChassisName   string
    LocalRouters       []LocalRouterInfo
    HasLocalRouters    bool
    FIPs               []string
    SNATIPs            []string
    NATIPToRouterMAC   map[string]string
    DiscoveredNetworks []*net.IPNet
    AllChassisNames    map[string]bool
}
```

Derived state from OVN databases.

### Type: LocalRouterInfo

```go
type LocalRouterInfo struct {
    RouterName  string
    RouterUUID  string
    LRPName     string
    LRPUUID     string
    LRPMAC      string
    LRPNetworks []string
    CRPort      string
}
```

Information about a logical router whose gateway is active on this chassis.

---

## Routing Package

### Type: RouteManager

```go
type RouteManager struct {
    bridgeDev            string
    vrfName              string
    vethNexthop          string
    routeTableID         int
    ovsWrapper           []string
    dryRun               bool
    vethLeakEnabled      bool
    vethProviderIP       string
    vethLeakTableID      int
    vethLeakRulePriority int
    networkFilters       []*net.IPNet
    frrPrefixList        string
    portForwardEnabled   bool
    portForwardDev       string
    portForwardTableID   int
    portForwardL3mdevAccept bool
    portForwardCTZone    int
    portForwards         []PortForwardVIP
    cachedPatchPort      string
    cachedOfport         string
    cachedBridgeMAC      string
}
```

Handles kernel routes on the provider bridge and FRR static routes.

#### `NewRouteManager(cfg Config) *RouteManager`
```go
func NewRouteManager(cfg Config) *RouteManager
```
Creates a new RouteManager instance.

**Parameters:**
- `cfg`: Configuration

**Returns:**
- `*RouteManager`: New route manager instance

#### `CheckBridgeDevice() error`
```go
func (rm *RouteManager) CheckBridgeDevice() error
```
Verifies that the bridge device exists and that the agent has sufficient privileges.

**Parameters:** None

**Returns:**
- `error`: Error if bridge device check fails

#### `EnsureBridgeIP(ip string) error`
```go
func (rm *RouteManager) EnsureBridgeIP(ip string) error
```
Adds a /32 IP address to the bridge device if not already present.

**Parameters:**
- `ip`: IP address to add

**Returns:**
- `error`: Error if IP addition fails

#### `RemoveBridgeIP(ip string) error`
```go
func (rm *RouteManager) RemoveBridgeIP(ip string) error
```
Removes the /32 IP address from the bridge device.

**Parameters:**
- `ip`: IP address to remove

**Returns:**
- `error`: Error if IP removal fails

#### `EnableProxyARP() error`
```go
func (rm *RouteManager) EnableProxyARP() error
```
Enables proxy ARP on the bridge device.

**Parameters:** None

**Returns:**
- `error`: Error if proxy ARP enable fails

#### `GetBridgeMAC() (net.HardwareAddr, error)`
```go
func (rm *RouteManager) GetBridgeMAC() (net.HardwareAddr, error)
```
Returns the hardware MAC address of the bridge device.

**Parameters:** None

**Returns:**
- `net.HardwareAddr`: MAC address
- `error`: Error if MAC retrieval fails

#### `AddKernelRoute(ip string) error`
```go
func (rm *RouteManager) AddKernelRoute(ip string) error
```
Adds a /32 kernel route for the specified IP.

**Parameters:**
- `ip`: IP address

**Returns:**
- `error`: Error if route addition fails

#### `DelKernelRoute(ip string) error`
```go
func (rm *RouteManager) DelKernelRoute(ip string) error
```
Removes a /32 kernel route for the specified IP.

**Parameters:**
- `ip`: IP address

**Returns:**
- `error`: Error if route removal fails

#### `ListKernelRoutes() ([]string, error)`
```go
func (rm *RouteManager) ListKernelRoutes() ([]string, error)
```
Returns all /32 routes on the bridge device.

**Parameters:** None

**Returns:**
- `[]string`: List of IP addresses
- `error`: Error if route listing fails

#### `AddFRRRoute(ip string) error`
```go
func (rm *RouteManager) AddFRRRoute(ip string) error
```
Adds a static /32 FRR route for the specified IP.

**Parameters:**
- `ip`: IP address

**Returns:**
- `error`: Error if route addition fails

#### `AddFRRRoutes(ips []string) error`
```go
func (rm *RouteManager) AddFRRRoutes(ips []string) error
```
Adds static /32 FRR routes for all specified IPs (batched).

**Parameters:**
- `ips`: List of IP addresses

**Returns:**
- `error`: Error if route addition fails

#### `DelFRRRoute(ip string) error`
```go
func (rm *RouteManager) DelFRRRoute(ip string) error
```
Removes a static /32 FRR route for the specified IP.

**Parameters:**
- `ip`: IP address

**Returns:**
- `error`: Error if route removal fails

#### `DelFRRRoutes(ips []string) error`
```go
func (rm *RouteManager) DelFRRRoutes(ips []string) error
```
Removes static /32 FRR routes for all specified IPs (batched).

**Parameters:**
- `ips`: List of IP addresses

**Returns:**
- `error`: Error if route removal fails

#### `HasFRRRoute(ip string) bool`
```go
func (rm *RouteManager) HasFRRRoute(ip string) bool
```
Checks if a static route for the IP exists in the VRF.

**Parameters:**
- `ip`: IP address

**Returns:**
- `bool`: True if route exists

#### `ListFRRRoutes() ([]string, error)`
```go
func (rm *RouteManager) ListFRRRoutes() ([]string, error)
```
Returns all static /32 routes in the VRF.

**Parameters:** None

**Returns:**
- `[]string`: List of IP addresses
- `error`: Error if route listing fails

#### `EnsureOVSFlows() error`
```go
func (rm *RouteManager) EnsureOVSFlows() error
```
Installs MAC-tweak flows on the bridge device.

**Parameters:** None

**Returns:**
- `error`: Error if flow installation fails

#### `ReconcileOVSHairpinFlows(ipToRouterMAC map[string]string) error`
```go
func (rm *RouteManager) ReconcileOVSHairpinFlows(ipToRouterMAC map[string]string) error
```
Installs per-IP hairpin flows on the bridge device.

**Parameters:**
- `ipToRouterMAC`: Map of IP to router MAC

**Returns:**
- `error`: Error if flow installation fails

#### `RemoveOVSFlows() error`
```go
func (rm *RouteManager) RemoveOVSFlows() error
```
Removes all agent-managed OVS flows from the bridge device.

**Parameters:** None

**Returns:**
- `error`: Error if flow removal fails

#### `SetupVethLeak() error`
```go
func (rm *RouteManager) SetupVethLeak() error
```
Creates a veth pair for selective route leaking between the default VRF and the provider VRF.

**Parameters:** None

**Returns:**
- `error`: Error if veth leak setup fails

#### `TeardownVethLeak() error`
```go
func (rm *RouteManager) TeardownVethLeak() error
```
Removes all veth leak resources.

**Parameters:** None

**Returns:**
- `error`: Error if veth leak teardown fails

#### `ReconcileVethLeakNetworks(desired []*net.IPNet) error`
```go
func (rm *RouteManager) ReconcileVethLeakNetworks(desired []*net.IPNet) error
```
Ensures per-network VRF routes and policy rules match the desired set of networks.

**Parameters:**
- `desired`: List of desired networks

**Returns:**
- `error`: Error if reconciliation fails

#### `SetupPortForward() error`
```go
func (rm *RouteManager) SetupPortForward() error
```
Sets up port forwarding (DNAT) rules.

**Parameters:** None

**Returns:**
- `error`: Error if port forwarding setup fails

#### `TeardownPortForward() error`
```go
func (rm *RouteManager) TeardownPortForward() error
```
Removes all port forwarding rules.

**Parameters:** None

**Returns:**
- `error`: Error if port forwarding teardown fails

#### `CleanupRoutingTable() error`
```go
func (rm *RouteManager) CleanupRoutingTable() error
```
Removes all routes and ip rules from the dedicated routing table.

**Parameters:** None

**Returns:**
- `error`: Error if cleanup fails

---

## OVS Package

### Functions

#### `MACTweakFlow(cookie, ofport, mac string, ipv6 bool) string`
```go
func MACTweakFlow(cookie, ofport, mac string, ipv6 bool) string
```
Returns the OpenFlow rule string for a MAC-tweak flow.

**Parameters:**
- `cookie`: Flow cookie identifier
- `ofport`: OpenFlow port number
- `mac`: MAC address to rewrite to
- `ipv6`: True for IPv6, false for IPv4

**Returns:**
- `string`: OpenFlow rule string

**Example:**
```go
flow := MACTweakFlow("0x999", "1", "aa:bb:cc:dd:ee:ff", false)
// Returns: "cookie=0x999,priority=900,ip,in_port=1,actions=mod_dl_dst:aa:bb:cc:dd:ee:ff,NORMAL"
```

#### `HairpinFlow(cookie, ofport, ip, bridgeMAC, routerMAC string, ipv6 bool) string`
```go
func HairpinFlow(cookie, ofport, ip, bridgeMAC, routerMAC string, ipv6 bool) string
```
Returns the OpenFlow rule string for a same-chassis hairpin flow.

**Parameters:**
- `cookie`: Flow cookie identifier
- `ofport`: OpenFlow port number
- `ip`: IP address to match
- `bridgeMAC`: Bridge MAC address
- `routerMAC`: Router port MAC address
- `ipv6`: True for IPv6, false for IPv4

**Returns:**
- `string`: OpenFlow rule string

**Example:**
```go
flow := HairpinFlow("0x998", "1", "203.0.113.10", "aa:bb:cc:dd:ee:ff", "11:22:33:44:55:66", false)
// Returns: "cookie=0x998,priority=910,ip,in_port=1,ip_dst=203.0.113.10/32,actions=mod_dl_src:aa:bb:cc:dd:ee:ff,mod_dl_dst:11:22:33:44:55:66,output:in_port"
```

---

## nftables Package

### Functions

#### `buildNftRuleset(forwards []PortForwardVIP, providerNetworks []*net.IPNet, ctZone int) string`
```go
func buildNftRuleset(forwards []PortForwardVIP, providerNetworks []*net.IPNet, ctZone int) string
```
Generates the complete nftables ruleset for port forwarding.

**Parameters:**
- `forwards`: List of port forwarding VIPs
- `providerNetworks`: List of provider networks
- `ctZone`: Conntrack zone number

**Returns:**
- `string`: Complete nftables ruleset

**Behavior:**
Generates up to eight chains:
1. `prerouting_ctzone`: Assigns conntrack zone
2. `output_ctzone`: Zone for locally generated packets
3. `prerouting_dnat`: DNAT rules
4. `prerouting_fwmark`: Policy routing marks
5. `output_fwmark`: Reply direction marks
6. `forward_veth_guard`: Veth return path security
7. `postrouting_fwmark_clear`: Clear reply marks
8. `postrouting_snat`: Masquerade rules

---

## Config Package

### Type: Config

```go
type Config struct {
    OVNSBRemote       string
    OVNNBRemote       string
    BridgeDev         string
    VRFName           string
    VethNexthop       string
    NetworkCIDRs      []string
    NetworkFilters    []*net.IPNet
    GatewayPort       string
    RouteTableID      int
    BridgeIP          string
    OVSWrapper        string
    ReconcileInterval time.Duration
    LogLevel          string
    DryRun            bool
    CleanupOnShutdown bool
    DrainOnShutdown   bool
    DrainTimeout      time.Duration
    FRRPrefixList     string
    StaleChassisGracePeriod time.Duration
    VethLeakEnabled      bool
    VethProviderIP       string
    VethLeakTableID      int
    VethLeakRulePriority int
    PortForwardEnabled      bool
    PortForwardDev          string
    PortForwardTableID      int
    PortForwardL3mdevAccept bool
    PortForwardCTZone       int
    PortForwards            []PortForwardVIP
}
```

Runtime configuration for the agent.

### Type: PortForwardVIP

```go
type PortForwardVIP struct {
    VIP               string
    ManageVIP         bool
    Masquerade        bool
    HairpinMasquerade bool
    Rules             []PortForwardRule
}
```

VIP with forwarding rules.

### Type: PortForwardRule

```go
type PortForwardRule struct {
    Proto      string
    Port       int
    DestAddr   string
    DestAddrs  []string
    DestPort   int
    Masquerade *bool
}
```

Single DNAT port forwarding rule.

#### `effectiveMasquerade(vipDefault bool) bool`
```go
func (r *PortForwardRule) effectiveMasquerade(vipDefault bool) bool
```
Returns the masquerade setting for this rule.

**Parameters:**
- `vipDefault`: VIP-level default masquerade setting

**Returns:**
- `bool`: Effective masquerade setting

#### `destAddrs() []string`
```go
func (r *PortForwardRule) destAddrs() []string
```
Returns the effective list of backend addresses for the rule.

**Parameters:** None

**Returns:**
- `[]string`: List of backend addresses

---

## Helper Functions

### `effectiveNetworkFilters(manual, discovered []*net.IPNet) []*net.IPNet`
```go
func effectiveNetworkFilters(manual, discovered []*net.IPNet) []*net.IPNet
```
Returns manual config if non-empty, otherwise discovered networks.

**Parameters:**
- `manual`: Manually configured networks
- `discovered`: Auto-discovered networks

**Returns:**
- `[]*net.IPNet`: Effective network filters

### `containedInAny(ip net.IP, nets []*net.IPNet) bool`
```go
func containedInAny(ip net.IP, nets []*net.IPNet) bool
```
Returns true if ip is contained in any of the given networks.

**Parameters:**
- `ip`: IP address to check
- `nets`: List of networks

**Returns:**
- `bool`: True if IP is in any network

### `uniqueNetworks(nets []*net.IPNet) []*net.IPNet`
```go
func uniqueNetworks(nets []*net.IPNet) []*net.IPNet
```
Deduplicates and sorts a list of *net.IPNet by string representation.

**Parameters:**
- `nets`: List of networks

**Returns:**
- `[]*net.IPNet`: Deduplicated and sorted list

### `parseNatAddressIPs(natAddr string) []string`
```go
func parseNatAddressIPs(natAddr string) []string
```
Extracts IP addresses from an OVN NatAddresses entry.

**Parameters:**
- `natAddr`: NatAddresses string

**Returns:**
- `[]string`: List of IP addresses

**Format:** "MAC IP1 [IP2 ...] [is_chassis_resident("port")]"

### `getHostname() (string, error)`
```go
func getHostname() (string, error)
```
Returns the hostname without domain.

**Parameters:** None

**Returns:**
- `string`: Hostname
- `error`: Error if hostname retrieval fails

---

## Constants

### Debounce Intervals

```go
const eventDebounceInterval = 500 * time.Millisecond
const reconcileDebounceInterval = 100 * time.Millisecond
```

### OVS Flow Cookies

```go
const ovsCookieMACTweak = "0x999"
const ovsCookieHairpin  = "0x998"
```

### OVS Flow Priorities

```go
// Hairpin flows (priority 910) fire before MAC-tweak (priority 900)
```

### nftables Constants

```go
const nftTableName       = "ovn-network-agent"
const dnatFwmark         = 0x100
const dnatReplyFwmark    = 0x200
const dnatFwmarkPriority = 150
const dnatReplyPriority  = 151
const dnatCTZoneDefault  = 64000
```

### Veth Leak Constants

```go
const vethDefaultName  = "veth-default"
const vethProviderName = "veth-provider"
const vethPrefixLen    = 30
const rtProtoOVNNetworkAgent = 44
```

### Gateway Chassis Constants

```go
const minActivePriority = 2
const maxStaleCleanupJitter = 30 * time.Second
```

### FRR Constants

```go
const frrBatchSize = 500
```

---

## Error Handling

### Common Errors

- **Configuration Error**: Invalid configuration values
- **Connection Error**: Failed to connect to OVN databases
- **Bridge Error**: Bridge device not found or inaccessible
- **Route Error**: Failed to add/remove routes
- **OVS Error**: Failed to install/remove OVS flows
- **nftables Error**: Failed to install/remove nftables rules

### Error Handling Pattern

```go
if err != nil {
    slog.Error("operation failed", "error", err)
    return err
}
```

---

## Usage Examples

### Basic Usage

```go
package main

import (
    "context"
    "log/slog"
    "os"
    "os/signal"
    "syscall"
)

func main() {
    // Load configuration
    cfg, err := loadConfig(os.Args[1:])
    if err != nil {
        slog.Error("configuration error", "error", err)
        os.Exit(1)
    }

    // Create agent
    agent, err := NewAgent(cfg)
    if err != nil {
        slog.Error("failed to create agent", "error", err)
        os.Exit(1)
    }

    // Setup context for cancellation
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Handle signals
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

    go func() {
        sig := <-sigCh
        slog.Info("received signal, shutting down", "signal", sig)
        cancel()
    }()

    // Run agent
    if err := agent.Run(ctx); err != nil && ctx.Err() == nil {
        slog.Error("agent exited with error", "error", err)
        os.Exit(1)
    }

    slog.Info("agent stopped")
}
```

### Custom Route Management

```go
// Add a kernel route
err := routeManager.AddKernelRoute("203.0.113.10")
if err != nil {
    slog.Error("failed to add kernel route", "error", err)
}

// Add FRR routes in batch
ips := []string{"203.0.113.10", "203.0.113.11", "203.0.113.12"}
err = routeManager.AddFRRRoutes(ips)
if err != nil {
    slog.Error("failed to add FRR routes", "error", err)
}

// List current routes
kernelRoutes, err := routeManager.ListKernelRoutes()
if err != nil {
    slog.Error("failed to list kernel routes", "error", err)
}
slog.Info("current kernel routes", "routes", kernelRoutes)
```

### OVN State Query

```go
// Get current state
state := ovnClient.GetState()

// Check if we have local routers
if state.HasLocalRouters {
    slog.Info("local routers active", "count", len(state.LocalRouters))
    
    // List FIPs
    slog.Info("floating IPs", "fips", state.FIPs)
    
    // List SNAT IPs
    slog.Info("SNAT IPs", "snat_ips", state.SNATIPs)
    
    // List discovered networks
    for _, network := range state.DiscoveredNetworks {
        slog.Info("discovered network", "network", network.String())
    }
}
```

### Port Forwarding Configuration

```yaml
port_forwards:
  - vip: "203.0.113.10"
    manage_vip: true
    masquerade: false
    hairpin_masquerade: true
    rules:
      - proto: "tcp"
        port: 80
        dest_addrs: ["10.0.0.10", "10.0.0.11"]
        dest_port: 8080
      - proto: "tcp"
        port: 443
        dest_addr: "10.0.0.10"
        dest_port: 8443
```

---

## Best Practices

1. **Always check errors**: All operations can fail
2. **Use dry-run mode**: Test configuration changes without applying them
3. **Monitor logs**: Use debug logging for troubleshooting
4. **Graceful shutdown**: Handle SIGINT/SIGTERM properly
5. **Idempotent operations**: All operations are safe to run multiple times
6. **Debouncing**: Rapid changes are coalesced automatically
7. **State verification**: Post-change verification ensures correctness

---

## Performance Considerations

1. **Batch operations**: FRR routes are batched (500 per call)
2. **Debouncing**: Rapid changes are coalesced
3. **Efficient queries**: OVSDB monitoring with incremental updates
4. **Caching**: OVS discovery results are cached
5. **State snapshots**: Immutable copies prevent race conditions

---

## Security Considerations

1. **Run as root**: Required for CAP_NET_ADMIN
2. **Secure OVSDB connections**: Use TLS/SSL
3. **Protect configuration**: Config file contains sensitive data
4. **nftables rules**: Affect system-wide packet processing
5. **VRF access**: Can modify VRF routing tables
