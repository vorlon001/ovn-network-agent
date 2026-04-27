# OVN Network Agent - Architecture Documentation

## Table of Contents
1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Component Architecture](#component-architecture)
4. [Data Flow](#data-flow)
5. [State Management](#state-management)
6. [Event Handling](#event-handling)
7. [Reconciliation](#reconciliation)
8. [Routing Architecture](#routing-architecture)
9. [Port Forwarding](#port-forwarding)
10. [High Availability](#high-availability)

---

## Overview

The OVN Network Agent is an event-driven daemon that monitors OVN (Open Virtual Network) databases and synchronizes network state between OVN and the local system. It's designed for OpenStack environments using OVN for networking.

### Key Responsibilities

1. **Monitor OVN Databases**: Watch Southbound (SB) and Northbound (NB) databases for changes
2. **Route Synchronization**: Manage Floating IP (FIP) and SNAT routes
3. **Gateway Management**: Handle gateway chassis failover and routing
4. **Port Forwarding**: Support DNAT for anycast VIPs
5. **VRF Integration**: Work with VRFs and FRR for BGP route announcement

### Design Principles

- **Event-Driven**: Reacts to OVSDB events in real-time
- **Idempotent**: All operations are safe to run multiple times
- **Debounced**: Coalesces rapid changes to avoid thrashing
- **Graceful Degradation**: Handles failures and partial states

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         OVN Network Agent                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐        │
│  │   Config     │    │    Agent     │    │  OVN Client  │        │
│  │   Manager    │◄───┤   Orchestr   │◄───┤   Wrapper    │        │
│  └──────────────┘    └──────────────┘    └──────────────┘        │
│         │                   │                   │                │
│         │                   │                   │                │
│         ▼                   ▼                   ▼                │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐        │
│  │  Route       │    │   State      │    │  OVSDB       │        │
│  │  Manager     │    │   Cache      │    │  Monitor     │        │
│  └──────────────┘    └──────────────┘    └──────────────┘        │
│         │                   │                   │                │
│         │                   │                   │                │
│         ▼                   ▼                   ▼                │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐        │
│  │  Kernel      │    │  FRR         │    │  OVS         │        │
│  │  Routes      │    │  Routes      │    │  Flows       │        │
│  └──────────────┘    └──────────────┘    └──────────────┘        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### External Dependencies

- **OVN Southbound DB**: Port bindings, chassis information
- **OVN Northbound DB**: NAT entries, logical routers, static routes
- **Linux Kernel**: Routes, rules, ARP, network interfaces
- **FRR**: BGP route announcement via vtysh
- **OVS**: OpenFlow flows on provider bridge
- **nftables**: Port forwarding DNAT rules

---

## Component Architecture

### 1. Main Entry Point ([`main.go`](../main.go))

**Responsibilities**:
- Configuration loading and validation
- Logging setup
- Signal handling (SIGINT, SIGTERM)
- Agent lifecycle management

**Key Functions**:
- `main()`: Entry point, loads config, creates agent, runs event loop
- `loadConfig()`: Merges CLI flags, env vars, and config file
- `setupLogging()`: Configures structured logging

### 2. Agent Orchestrator ([`agent.go`](../agent.go))

**Responsibilities**:
- Main event loop and reconciliation
- Bridge device setup
- Veth leak and port forwarding setup
- Stale chassis cleanup
- Gateway drain on shutdown

**Key Structures**:
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

**Key Methods**:
- `NewAgent()`: Creates new agent instance
- `Run()`: Main event loop with reconciliation
- `reconcile()`: Ensures local state matches OVN state
- `cleanup()`: Removes all managed resources
- `triggerReconcile()`: Non-blocking reconcile trigger

### 3. OVN Client ([`ovn.go`](../ovn.go))

**Responsibilities**:
- OVSDB connection management
- Database monitoring and event handling
- State refresh and caching
- OVN NB operations (routes, MAC bindings)

**Key Structures**:
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

**Key Methods**:
- `Connect()`: Connects to SB and NB databases
- `refreshState()`: Refreshes state from both databases
- `GetState()`: Returns state snapshot
- `EnsureGatewayRouting()`: Sets up default routes and MAC bindings
- `EnsureActivePriorityLead()`: Maintains HA priority lead

### 4. Route Manager ([`routing.go`](../routing.go))

**Responsibilities**:
- Kernel route management (via netlink)
- FRR route management (via vtysh)
- OVS flow management
- Veth leak setup and reconciliation
- Port forwarding setup

**Key Structures**:
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

**Key Methods**:
- `AddKernelRoute()` / `DelKernelRoute()`: Kernel route operations
- `AddFRRRoute()` / `DelFRRRoute()`: FRR route operations
- `EnsureOVSFlows()`: Installs MAC-tweak flows
- `ReconcileOVSHairpinFlows()`: Manages hairpin flows
- `SetupVethLeak()`: Creates veth pair for route leaking
- `ReconcileVethLeakNetworks()`: Manages per-network leak routes
- `SetupPortForward()`: Configures DNAT rules

### 5. OVS Flow Manager ([`ovs.go`](../ovs.go))

**Responsibilities**:
- OpenFlow flow installation and removal
- Patch port discovery
- Bridge MAC address retrieval

**Key Flow Types**:
- **MAC-tweak flows** (priority 900): Rewrite destination MAC to bridge MAC
- **Hairpin flows** (priority 910): Reflect same-chassis traffic back to OVN

### 6. nftables Manager ([`nftables.go`](../nftables.go))

**Responsibilities**:
- Port forwarding DNAT rules
- Conntrack zone management
- Policy routing with fwmarks
- Masquerade rules

**Key Chains**:
- `prerouting_ctzone`: Assign conntrack zone
- `output_ctzone`: Zone for locally generated packets
- `prerouting_dnat`: DNAT rules
- `prerouting_fwmark`: Policy routing marks
- `output_fwmark`: Reply direction marks
- `forward_veth_guard`: Veth return path security
- `postrouting_fwmark_clear`: Clear reply marks
- `postrouting_snat`: Masquerade rules

### 7. Gateway Manager ([`ovn_gateway.go`](../ovn_gateway.go))

**Responsibilities**:
- Default route management in OVN NB
- Static MAC binding management
- Gateway chassis priority management
- Stale chassis cleanup

**Key Methods**:
- `EnsureGatewayRouting()`: Ensures default routes and MAC bindings
- `EnsureActivePriorityLead()`: Maintains HA priority lead
- `DrainGateways()`: Lowers priority for graceful shutdown
- `RestoreDrainedGateways()`: Restores priority after restart
- `CleanupStaleChassisEntries()`: Removes entries from dead chassis

---

## Data Flow

### Startup Flow

```
main()
  │
  ├─► loadConfig()
  │     └─► Parse CLI flags, env vars, config file
  │
  ├─► setupLogging()
  │
  ├─► NewAgent()
  │     ├─► NewRouteManager()
  │     └─► NewOVNClient()
  │
  └─► agent.Run()
        │
        ├─► CheckBridgeDevice()
        ├─► EnsureBridgeIP()
        ├─► EnableProxyARP()
        ├─► SetupVethLeak()
        ├─► SetupPortForward()
        │
        ├─► ovn.Connect()
        │     ├─► Connect to SB DB
        │     ├─► Monitor SB tables
        │     ├─► Connect to NB DB
        │     ├─► Monitor NB tables
        │     └─► refreshState()
        │
        ├─► RestoreDrainedGateways()
        │
        ├─► reconcile() [initial]
        │
        └─► Event Loop
              ├─► reconcileCh (event-triggered)
              ├─► ticker.C (periodic)
              └─► ctx.Done() (shutdown)
```

### Reconciliation Flow

```
reconcile()
  │
  ├─► ovn.GetState()
  │     └─► Returns state snapshot
  │
  ├─► computeEffectiveNetworks()
  │     └─► Manual config or auto-discovered
  │
  ├─► Build hairpinMACMap
  │     └─► NAT IPs → Router MACs
  │
  ├─► ovn.EnsureGatewayRouting()
  │     ├─► ensureDefaultRoute()
  │     └─► ensureStaticMACBinding()
  │
  ├─► ovn.EnsureActivePriorityLead()
  │     └─► Boost priority above peers
  │
  ├─► routing.EnsureOVSFlows()
  │     └─► MAC-tweak flows
  │
  ├─► routing.ReconcileOVSHairpinFlows()
  │     └─► Per-IP hairpin flows
  │
  ├─► Reconcile kernel routes
  │     ├─► List current routes
  │     ├─► Add missing routes
  │     └─► Remove stale routes
  │
  ├─► Reconcile FRR routes
  │     ├─► List current routes
  │     ├─► Add missing routes
  │     └─► Remove stale routes
  │
  ├─► routing.ReconcileVethLeakNetworks()
  │     ├─► Add missing network routes
  │     └─► Remove stale network routes
  │
  ├─► Reconcile FRR prefix-list
  │     └─► Update with discovered networks
  │
  └─► Check stale chassis
        └─► Cleanup after grace period
```

### Event Handling Flow

```
OVSDB Event
  │
  ├─► SB Event (Port_Binding, Chassis)
  │     ├─► chassisredirect? → immediateStateRefresh()
  │     └─► Other → debounceStateRefresh()
  │
  └─► NB Event (NAT, Logical_Router, etc.)
        └─► debounceStateRefresh()
              │
              ├─► Wait 500ms (coalesce)
              │
              └─► refreshState()
                    │
                    ├─► List SB Port_Binding
                    ├─► List SB Chassis
                    ├─► List NB Logical_Router_Port
                    ├─► List NB Logical_Router
                    ├─► List NB NAT
                    │
                    └─► Update OVNState
                          │
                          └─► scheduleReconcile()
                                │
                                └─► Wait 100ms
                                      │
                                      └─► onChange()
                                            └─► triggerReconcile()
```

---

## State Management

### OVN State

The agent maintains a cached view of OVN state in [`OVNState`](../ovn.go:156):

```go
type OVNState struct {
    mu                 sync.RWMutex
    LocalChassisName   string              // Local hostname
    LocalRouters       []LocalRouterInfo   // Active routers
    HasLocalRouters    bool
    FIPs               []string            // Floating IPs
    SNATIPs            []string            // SNAT IPs
    NATIPToRouterMAC   map[string]string   // IP → Router MAC
    DiscoveredNetworks []*net.IPNet        // Auto-discovered CIDRs
    AllChassisNames    map[string]bool     // All chassis in SB
}
```

### State Refresh Process

1. **Query SB Port_Binding**: Find chassisredirect ports
2. **Query SB Chassis**: Build chassis hostname map
3. **Query NB Logical_Router_Port**: Build LRP name → UUID map
4. **Query NB Logical_Router**: Find routers with local LRPs
5. **Query NB NAT**: Filter to locally-active routers
6. **Extract SNAT IPs from SB**: Get IPs from gateway patch ports
7. **Update state atomically**: Lock, update, unlock

### State Consistency

- **Thread-safe**: All state access protected by RWMutex
- **Snapshot-based**: [`GetState()`](../ovn.go:342) returns immutable copy
- **Debounced updates**: Rapid changes coalesced
- **Periodic reconciliation**: Safety net every 60s

---

## Event Handling

### Event Sources

1. **Southbound Events**:
   - `Port_Binding` table changes
   - `Chassis` table changes

2. **Northbound Events**:
   - `NAT` table changes
   - `Logical_Router` table changes
   - `Logical_Router_Port` table changes
   - `Logical_Router_Static_Route` table changes
   - `Static_MAC_Binding` table changes
   - `Gateway_Chassis` table changes

### Event Handlers

**SB Handler** ([`sbEventHandler`](../ovn.go:664)):
- Chassisredirect changes → Immediate refresh (fast failover)
- Other changes → Debounced refresh

**NB Handler** ([`nbEventHandler`](../ovn.go:714)):
- All relevant changes → Debounced refresh

### Debouncing Strategy

- **State refresh debounce**: 500ms after first event
- **Reconcile debounce**: 100ms after state refresh
- **Chassisredirect bypass**: No debounce for fast failover

---

## Reconciliation

### Reconciliation Triggers

1. **Event-triggered**: OVSDB changes
2. **Periodic**: Every `reconcile_interval` (default 60s)
3. **Manual**: Via `triggerReconcile()`

### Reconciliation Steps

1. **Get current state**: Snapshot from OVN
2. **Compute desired state**: Based on local routers
3. **Compare and sync**:
   - OVN NB routes and MAC bindings
   - Kernel routes
   - FRR routes
   - OVS flows
   - Veth leak routes
   - FRR prefix-list
4. **Verify**: Post-change verification
5. **Retry**: If verification fails

### Idempotency

All reconciliation operations are idempotent:
- Routes: Use `RouteReplace` / `RouteDel` (no error if exists/absent)
- OVS flows: Delete before add
- FRR routes: Batch operations with validation
- nftables: Full table replace

---

## Routing Architecture

### Multi-Layer Routing

```
┌─────────────────────────────────────────────────────────┐
│                    Incoming Packet                       │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  1. OVS Flow Processing (br-ex)                          │
│     - Hairpin flow (priority 910) → Reflect to OVN       │
│     - MAC-tweak flow (priority 900) → Rewrite MAC        │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  2. Kernel Routing (ip route)                           │
│     - /32 routes to br-ex (main or custom table)        │
│     - Policy rules for table lookup                     │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  3. VRF Routing (vrf-provider)                          │
│     - FRR static routes via veth-provider              │
│     - BGP announcement to external peers                │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  4. Veth Leak (optional)                                │
│     - Default VRF ↔ Provider VRF route leaking          │
│     - Per-network policy rules                         │
└─────────────────────────────────────────────────────────┘
```

### Route Types

1. **Kernel Routes** ([`routing_linux.go`](../routing_linux.go:139)):
   - `/32` routes on `br-ex`
   - Managed via netlink
   - Optional custom routing table

2. **FRR Routes** ([`routing.go`](../routing.go:99)):
   - `/32` static routes in VRF
   - Managed via vtysh
   - Announced via BGP

3. **Veth Leak Routes** ([`routing_linux.go`](../routing_linux.go:574)):
   - Per-network routes via veth pair
   - Policy rules for selective leaking
   - Custom protocol tag (44)

### Route Synchronization

All route types are synchronized independently:
- Each type has its own desired/actual comparison
- Missing routes are added
- Stale routes are removed
- Verification after changes

---

## Port Forwarding

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│  External Client → VIP (anycast)                        │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  1. nftables prerouting_ctzone                          │
│     - Assign conntrack zone (shared)                    │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  2. nftables prerouting_dnat                            │
│     - Single backend: Direct DNAT                       │
│     - Multiple backends: jhash load balancing           │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  3. nftables prerouting_fwmark                          │
│     - Mark original: 0x100 → main table                 │
│     - Mark reply: 0x200 → VRF table                     │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  4. Policy Routing (ip rule)                            │
│     - fwmark 0x100 → lookup main                        │
│     - fwmark 0x200 → lookup VRF                         │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  5. Backend Processing                                  │
│     - Local backend: OUTPUT chains                      │
│     - Remote backend: VRF routing                       │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  6. Return Path                                         │
│     - nftables output_fwmark (local)                   │
│     - nftables postrouting_snat (masquerade)           │
│     - nftables forward_veth_guard (security)           │
└─────────────────────────────────────────────────────────┘
```

### Key Features

1. **Sticky Load Balancing**: jhash on source IP
2. **Cross-VRF Support**: Conntrack zone sharing
3. **Local Backend Support**: OUTPUT chains for same-host
4. **Masquerade Control**: Per-rule and VIP-level
5. **Hairpin Masquerade**: Source-selective SNAT
6. **Return Path Security**: Veth guard chain

### Configuration

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
```

---

## High Availability

### Gateway Chassis Failover

```
┌─────────────────────────────────────────────────────────┐
│  Active Chassis (priority 2)                            │
│  ┌─────────────────────────────────────────────────┐   │
│  │ chassisredirect: cr-lrp-abc123                  │   │
│  │ chassis: node1 (priority 2)                     │   │
│  │ status: ACTIVE                                  │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                          │
                          │ Failover
                          ▼
┌─────────────────────────────────────────────────────────┐
│  Standby Chassis (priority 1)                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │ chassisredirect: cr-lrp-abc123                  │   │
│  │ chassis: node2 (priority 1)                     │   │
│  │ status: STANDBY → ACTIVE                        │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Failover Detection

1. **SB Port_Binding change**: Chassis field updates
2. **Immediate refresh**: Bypass debounce for fast reaction
3. **State update**: New local routers detected
4. **Reconciliation**: Routes and flows updated

### Graceful Shutdown

1. **Drain mode**: Lower priority to 0
2. **Wait for migration**: ovn-northd moves to standby
3. **Cleanup**: Remove routes and flows
4. **Exit**: Clean shutdown

### Stale Chassis Cleanup

1. **Detect missing chassis**: Not in SB Chassis table
2. **Wait grace period**: Default 5 minutes
3. **Cleanup NB entries**: Remove routes and MAC bindings
4. **Random jitter**: 0-30s to prevent thundering herd

---

## Configuration

### Configuration Priority

```
CLI Flags > Environment Variables > Config File > Defaults
```

### Key Configuration Options

| Option | Default | Description |
|--------|---------|-------------|
| `ovn_sb_remote` | Required | OVN SB DB endpoints |
| `ovn_nb_remote` | Required | OVN NB DB endpoints |
| `bridge_dev` | `br-ex` | Provider bridge device |
| `vrf_name` | `vrf-provider` | VRF for FRR routes |
| `veth_nexthop` | `169.254.0.1` | Nexthop for FRR routes |
| `route_table_id` | `0` | Kernel route table (0 = main) |
| `bridge_ip` | `169.254.169.254` | Link-local IP on bridge |
| `reconcile_interval` | `60s` | Periodic reconciliation |
| `dry_run` | `false` | Log only, no changes |
| `cleanup_on_shutdown` | `true` | Remove routes on shutdown |
| `drain_on_shutdown` | `true` | Drain gateways before shutdown |
| `drain_timeout` | `60s` | Max drain wait time |
| `stale_chassis_grace_period` | `5m` | Grace period for cleanup |
| `veth_leak_enabled` | `true` | Enable veth VRF leak |
| `port_forward_dev` | `loopback1` | Loopback for VIPs |

---

## Dependencies

### Go Modules

- `github.com/ovn-kubernetes/libovsdb`: OVSDB client library
- `github.com/vishvananda/netlink`: Linux networking
- `github.com/cenkalti/backoff/v4`: Exponential backoff
- `gopkg.in/yaml.v3`: YAML configuration

### System Dependencies

- `ovs-vsctl` / `ovs-ofctl`: OVS management
- `vtysh`: FRR configuration
- `nft`: nftables management
- `ip`: Network configuration (via netlink)

---

## Security Considerations

1. **Privilege Requirements**: Must run as root (CAP_NET_ADMIN)
2. **Network Exposure**: Listens on OVSDB connections
3. **Configuration**: Sensitive data in config file (DB endpoints)
4. **nftables**: Rules affect system-wide packet processing
5. **VRF Access**: Can modify VRF routing tables

---

## Performance

### Scalability

- **Route Count**: Tested with thousands of FIPs
- **Batch Operations**: FRR routes batched (500 per call)
- **Debouncing**: Coalesces rapid changes
- **Efficient Queries**: OVSDB monitoring with incremental updates

### Resource Usage

- **Memory**: Proportional to route count
- **CPU**: Event-driven, low idle usage
- **Network**: OVSDB monitoring traffic only

---

## Troubleshooting

### Common Issues

1. **Routes not appearing**: Check `dry_run` mode
2. **OVN connection failed**: Verify DB endpoints
3. **Bridge device missing**: Ensure `br-ex` exists
4. **Permission denied**: Run as root
5. **FRR routes not syncing**: Check vtysh connectivity

### Debug Logging

Enable debug logging:
```bash
ovn-network-agent --log-level debug
```

### Dry Run Mode

Test without making changes:
```bash
ovn-network-agent --dry-run
```

---

## Future Enhancements

Potential areas for improvement:
1. IPv6 support (partial)
2. Metrics and monitoring
3. Dynamic configuration reload
4. Health check endpoints
5. Enhanced error recovery
6. Performance optimizations
