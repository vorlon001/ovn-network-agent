# OVN Network Agent - C4 Architecture

## Table of Contents
1. [C4 Model Overview](#c4-model-overview)
2. [Level 1: System Context](#level-1-system-context)
3. [Level 2: Container View](#level-2-container-view)
4. [Level 3: Component View](#level-3-component-view)
5. [Level 4: Code View](#level-4-code-view)

---

## C4 Model Overview

The C4 model is a simple way to model software architecture at different levels of abstraction:

- **Level 1: System Context** - Big picture view of the system and its environment
- **Level 2: Container View** - Containers (applications, data stores, etc.) within the system
- **Level 3: Component View** - Components within each container
- **Level 4: Code View** - Detailed design of key components

---

## Level 1: System Context

### System Context Diagram

```mermaid
C4Context
    title OVN Network Agent - System Context

    Person(user, "OpenStack Administrator", "Manages OpenStack networking")
    
    System(ovn_network_agent, "OVN Network Agent", "Event-driven network agent for OVN-based OpenStack environments")
    
    System_Ext(ovn_sb, "OVN Southbound DB", "OVN database for runtime state")
    System_Ext(ovn_nb, "OVN Northbound DB", "OVN database for logical network configuration")
    System_Ext(frr, "FRRouting", "BGP route announcement")
    System_Ext(ovs, "Open vSwitch", "OpenFlow switch and bridge management")
    System_Ext(kernel, "Linux Kernel", "Network stack, routes, rules")
    System_Ext(nftables, "nftables", "Packet filtering and NAT")
    
    Rel(user, ovn_network_agent, "Configures and monitors", "CLI flags, config file")
    Rel(ovn_network_agent, ovn_sb, "Monitors for changes", "OVSDB protocol")
    Rel(ovn_network_agent, ovn_nb, "Monitors and updates", "OVSDB protocol")
    Rel(ovn_network_agent, frr, "Manages static routes", "vtysh CLI")
    Rel(ovn_network_agent, ovs, "Installs OpenFlow flows", "ovs-ofctl CLI")
    Rel(ovn_network_agent, kernel, "Manages routes and rules", "netlink API")
    Rel(ovn_network_agent, nftables, "Installs DNAT rules", "nft CLI")
```

### System Context Description

**OVN Network Agent** is an event-driven daemon that synchronizes network state between OVN databases and the local system. It runs on each compute node in an OpenStack deployment using OVN for networking.

**External Systems:**

1. **OVN Southbound DB** - Contains runtime state including port bindings and chassis information
2. **OVN Northbound DB** - Contains logical network configuration including NAT entries and routers
3. **FRRouting** - BGP daemon for route announcement to external networks
4. **Open vSwitch** - Software switch providing OpenFlow capabilities
5. **Linux Kernel** - Provides network stack, routing, and packet processing
6. **nftables** - Packet filtering and NAT framework

**Key Interactions:**

- Monitors OVN databases for real-time changes
- Installs kernel routes for Floating IPs and SNAT IPs
- Manages FRR static routes for BGP announcement
- Installs OpenFlow flows on provider bridge
- Configures nftables for port forwarding (DNAT)

---

## Level 2: Container View

### Container Diagram

```mermaid
C4Container
    title OVN Network Agent - Container View

    Person(admin, "OpenStack Administrator")
    
    Container_Boundary(ovn_agent_boundary, "OVN Network Agent") {
        Container(main, "Main Process", "Go binary", "Entry point and lifecycle management")
        Container(agent, "Agent Orchestrator", "Go package", "Main event loop and reconciliation")
        Container(ovn_client, "OVN Client", "Go package", "OVSDB connection and monitoring")
        Container(route_manager, "Route Manager", "Go package", "Route and flow management")
        Container(config, "Config Manager", "Go package", "Configuration loading")
    }
    
    System_Ext(ovn_sb, "OVN Southbound DB")
    System_Ext(ovn_nb, "OVN Northbound DB")
    System_Ext(frr, "FRRouting")
    System_Ext(ovs, "Open vSwitch")
    System_Ext(kernel, "Linux Kernel")
    System_Ext(nftables, "nftables")
    
    Rel(admin, main, "Starts with config", "CLI flags, env vars, config file")
    Rel(main, config, "Loads configuration")
    Rel(main, agent, "Creates and runs")
    Rel(agent, ovn_client, "Queries state", "State snapshot")
    Rel(agent, route_manager, "Manages routes", "Add/remove operations")
    Rel(ovn_client, ovn_sb, "Monitors", "OVSDB")
    Rel(ovn_client, ovn_nb, "Monitors and updates", "OVSDB")
    Rel(route_manager, frr, "Manages routes", "vtysh")
    Rel(route_manager, ovs, "Installs flows", "ovs-ofctl")
    Rel(route_manager, kernel, "Manages routes", "netlink")
    Rel(route_manager, nftables, "Installs rules", "nft")
```

### Container Descriptions

#### Main Process
- **Technology**: Go binary
- **Responsibilities**:
  - Configuration loading and validation
  - Logging setup
  - Signal handling (SIGINT, SIGTERM)
  - Agent lifecycle management
  - Graceful shutdown coordination

#### Agent Orchestrator
- **Technology**: Go package ([`agent.go`](../agent.go))
- **Responsibilities**:
  - Main event loop with reconciliation
  - Bridge device setup
  - Veth leak and port forwarding setup
  - Stale chassis cleanup
  - Gateway drain on shutdown
  - Periodic reconciliation (default 60s)

#### OVN Client
- **Technology**: Go package ([`ovn.go`](../ovn.go))
- **Responsibilities**:
  - OVSDB connection management
  - Database monitoring and event handling
  - State refresh and caching
  - OVN NB operations (routes, MAC bindings)
  - Debouncing of rapid changes

#### Route Manager
- **Technology**: Go package ([`routing.go`](../routing.go))
- **Responsibilities**:
  - Kernel route management (via netlink)
  - FRR route management (via vtysh)
  - OVS flow management
  - Veth leak setup and reconciliation
  - Port forwarding setup

#### Config Manager
- **Technology**: Go package ([`config.go`](../config.go))
- **Responsibilities**:
  - Configuration loading from multiple sources
  - Priority: CLI flags > env vars > config file > defaults
  - Validation of configuration values
  - Port forwarding rule parsing

---

## Level 3: Component View

### Component Diagram - Agent Orchestrator

```mermaid
C4Component
    title Agent Orchestrator - Component View

    Component(agent, "Agent", "Main orchestrator")
    Component(reconciler, "Reconciler", "State synchronization")
    Component(drain_handler, "Drain Handler", "Graceful shutdown")
    Component(stale_cleanup, "Stale Cleanup", "Dead chassis cleanup")
    Component(veth_setup, "Veth Setup", "Route leak configuration")
    Component(pf_setup, "Port Forward Setup", "DNAT configuration")
    
    ComponentDb(state_cache, "State Cache", "OVN state snapshot")
    
    Rel(agent, reconciler, "Triggers", "Event or periodic")
    Rel(agent, drain_handler, "Calls on shutdown", "SIGINT/SIGTERM")
    Rel(agent, stale_cleanup, "Calls during reconcile", "Periodic check")
    Rel(agent, veth_setup, "Calls on startup", "Initial setup")
    Rel(agent, pf_setup, "Calls on startup", "Initial setup")
    Rel(reconciler, state_cache, "Reads", "State snapshot")
    Rel(drain_handler, state_cache, "Reads", "Local chassis name")
    Rel(stale_cleanup, state_cache, "Reads", "All chassis names")
```

### Component Diagram - OVN Client

```mermaid
C4Component
    title OVN Client - Component View

    Component(ovn_client, "OVNClient", "OVSDB wrapper")
    Component(sb_monitor, "SB Monitor", "Southbound monitoring")
    Component(nb_monitor, "NB Monitor", "Northbound monitoring")
    Component(state_refresher, "State Refresher", "State refresh logic")
    Component(event_handler, "Event Handler", "OVSDB event processing")
    Component(debouncer, "Debouncer", "Change coalescing")
    Component(nb_writer, "NB Writer", "Northbound writes")
    
    ComponentDb(sb_cache, "SB Cache", "Southbound table cache")
    ComponentDb(nb_cache, "NB Cache", "Northbound table cache")
    ComponentDb(ovn_state, "OVN State", "Derived state")
    
    Rel(ovn_client, sb_monitor, "Connects and monitors")
    Rel(ovn_client, nb_monitor, "Connects and monitors")
    Rel(sb_monitor, sb_cache, "Updates", "OVSDB events")
    Rel(nb_monitor, nb_cache, "Updates", "OVSDB events")
    Rel(sb_monitor, event_handler, "Notifies", "SB events")
    Rel(nb_monitor, event_handler, "Notifies", "NB events")
    Rel(event_handler, debouncer, "Triggers", "State refresh")
    Rel(debouncer, state_refresher, "Calls", "After debounce")
    Rel(state_refresher, sb_cache, "Queries", "Port_Binding, Chassis")
    Rel(state_refresher, nb_cache, "Queries", "NAT, Router, LRP, etc.")
    Rel(state_refresher, ovn_state, "Updates", "Derived state")
    Rel(nb_writer, nb_cache, "Writes", "Routes, MAC bindings")
```

### Component Diagram - Route Manager

```mermaid
C4Component
    title Route Manager - Component View

    Component(route_manager, "RouteManager", "Route orchestration")
    Component(kernel_routes, "Kernel Routes", "Linux routing table")
    Component(frr_routes, "FRR Routes", "BGP static routes")
    Component(ovs_flows, "OVS Flows", "OpenFlow rules")
    Component(veth_leak, "Veth Leak", "VRF route leaking")
    Component(port_forward, "Port Forward", "DNAT configuration")
    Component(prefix_list, "Prefix List", "FRR prefix-list")
    
    ComponentDb(route_cache, "Route Cache", "Current route state")
    ComponentDb(ovs_cache, "OVS Cache", "Patch port, MAC, ofport")
    
    Rel(route_manager, kernel_routes, "Manages", "Add/remove /32 routes")
    Rel(route_manager, frr_routes, "Manages", "Add/remove static routes")
    Rel(route_manager, ovs_flows, "Manages", "MAC-tweak, hairpin flows")
    Rel(route_manager, veth_leak, "Manages", "Veth pair, leak routes")
    Rel(route_manager, port_forward, "Manages", "nftables DNAT rules")
    Rel(route_manager, prefix_list, "Manages", "FRR prefix-list updates")
    Rel(kernel_routes, route_cache, "Caches", "Current routes")
    Rel(frr_routes, route_cache, "Caches", "Current routes")
    Rel(ovs_flows, ovs_cache, "Caches", "Discovery results")
```

### Component Descriptions

#### Agent Orchestrator Components

**Reconciler**
- Synchronizes local state with OVN state
- Compares desired vs actual routes
- Triggers add/remove operations
- Performs post-change verification

**Drain Handler**
- Lowers gateway chassis priority on shutdown
- Waits for migration to standby chassis
- Coordinates with graceful shutdown

**Stale Cleanup**
- Tracks missing chassis
- Waits for grace period (default 5m)
- Removes OVN NB entries from dead chassis
- Adds random jitter to prevent thundering herd

**Veth Setup**
- Creates veth pair for route leaking
- Configures VRF membership
- Sets up static neighbor entries
- Installs default route in leak table

**Port Forward Setup**
- Configures nftables DNAT rules
- Sets up conntrack zones
- Configures policy routing with fwmarks
- Installs masquerade rules

#### OVN Client Components

**SB Monitor**
- Connects to OVN Southbound DB
- Monitors Port_Binding and Chassis tables
- Detects chassisredirect changes (fast failover)
- Updates SB cache

**NB Monitor**
- Connects to OVN Northbound DB
- Monitors NAT, Logical_Router, LRP tables
- Updates NB cache
- Triggers state refresh on changes

**State Refresher**
- Queries both databases
- Builds derived state
- Maps chassisredirect → LRP → Router → NAT
- Extracts SNAT IPs from SB gateway ports
- Updates OVN state atomically

**Event Handler**
- Processes OVSDB events
- Distinguishes chassisredirect (immediate) vs other (debounced)
- Triggers state refresh

**Debouncer**
- Coalesces rapid changes
- 500ms debounce for state refresh
- 100ms debounce for reconciliation
- Bypasses debounce for chassisredirect

**NB Writer**
- Writes to OVN Northbound DB
- Manages default routes
- Manages static MAC bindings
- Manages gateway chassis priorities

#### Route Manager Components

**Kernel Routes**
- Manages /32 routes on br-ex
- Uses netlink API
- Supports custom routing tables
- Manages policy rules

**FRR Routes**
- Manages static routes in VRF
- Uses vtysh CLI
- Batched operations (500 per call)
- Announced via BGP

**OVS Flows**
- Installs MAC-tweak flows (priority 900)
- Installs hairpin flows (priority 910)
- Discovers patch port and ofport
- Caches bridge MAC

**Veth Leak**
- Creates veth pair (veth-default, veth-provider)
- Places veth-provider in VRF
- Manages per-network leak routes
- Manages policy rules

**Port Forward**
- Configures nftables DNAT rules
- Supports sticky load balancing (jhash)
- Manages conntrack zones
- Configures masquerade rules

**Prefix List**
- Updates FRR prefix-list
- Adds permit entries for discovered networks
- Format: `permit <network> ge 32 le 32`

---

## Level 4: Code View

### Code View - State Refresh

```mermaid
C4Component
    title State Refresh - Code View

    Component(refreshState, "refreshState()", "Main state refresh")
    Component(querySB, "Query SB", "Southbound queries")
    Component(queryNB, "Query NB", "Northbound queries")
    Component(buildState, "Build State", "State construction")
    Component(updateState, "Update State", "Atomic update")
    
    ComponentDb(sb_pb, "Port_Binding", "SB table")
    ComponentDb(sb_chassis, "Chassis", "SB table")
    ComponentDb(nb_lrp, "Logical_Router_Port", "NB table")
    ComponentDb(nb_router, "Logical_Router", "NB table")
    ComponentDb(nb_nat, "NAT", "NB table")
    ComponentDb(ovn_state, "OVNState", "In-memory state")
    
    Rel(refreshState, querySB, "Calls")
    Rel(querySB, sb_pb, "List()")
    Rel(querySB, sb_chassis, "List()")
    Rel(refreshState, queryNB, "Calls")
    Rel(queryNB, nb_lrp, "List()")
    Rel(queryNB, nb_router, "List()")
    Rel(queryNB, nb_nat, "List()")
    Rel(refreshState, buildState, "Calls")
    Rel(buildState, ovn_state, "Constructs")
    Rel(refreshState, updateState, "Calls")
    Rel(updateState, ovn_state, "Lock/Update/Unlock")
```

### Code View - Reconciliation

```mermaid
C4Component
    title Reconciliation - Code View

    Component(reconcile, "reconcile()", "Main reconcile")
    Component(getState, "GetState()", "State snapshot")
    Component(ovnOps, "OVN Operations", "NB writes")
    Component(ovsOps, "OVS Operations", "Flow management")
    Component(kernelOps, "Kernel Operations", "Route management")
    Component(frrOps, "FRR Operations", "Route management")
    Component(vethOps, "Veth Operations", "Leak management")
    Component(verify, "Verify", "Post-change check")
    
    ComponentDb(ovn_state, "OVNState", "State snapshot")
    ComponentDb(kernel_routes, "Kernel Routes", "Current routes")
    ComponentDb(frr_routes, "FRR Routes", "Current routes")
    ComponentDb(ovs_flows, "OVS Flows", "Current flows")
    
    Rel(reconcile, getState, "Calls")
    Rel(getState, ovn_state, "Returns snapshot")
    Rel(reconcile, ovnOps, "Calls")
    Rel(ovnOps, ovn_state, "Reads")
    Rel(reconcile, ovsOps, "Calls")
    Rel(ovsOps, ovs_flows, "Reads/Updates")
    Rel(reconcile, kernelOps, "Calls")
    Rel(kernelOps, kernel_routes, "Reads/Updates")
    Rel(reconcile, frrOps, "Calls")
    Rel(frrOps, frr_routes, "Reads/Updates")
    Rel(reconcile, vethOps, "Calls")
    Rel(reconcile, verify, "Calls")
    Rel(verify, kernel_routes, "Verifies")
    Rel(verify, frr_routes, "Verifies")
```

### Code View - Event Handling

```mermaid
C4Component
    title Event Handling - Code View

    Component(sbHandler, "sbEventHandler", "SB event handler")
    Component(nbHandler, "nbEventHandler", "NB event handler")
    Component(immediateRefresh, "immediateStateRefresh()", "Fast path")
    Component(debounceRefresh, "debounceStateRefresh()", "Debounced path")
    Component(stateTimer, "State Timer", "500ms debounce")
    Component(reconcileTimer, "Reconcile Timer", "100ms debounce")
    Component(refreshState, "refreshState()", "State refresh")
    Component(scheduleReconcile, "scheduleReconcile()", "Reconcile trigger")
    Component(onChange, "onChange()", "Callback")
    
    ComponentDb(ovn_state, "OVNState", "State")
    
    Rel(sbHandler, immediateRefresh, "Calls", "chassisredirect")
    Rel(sbHandler, debounceRefresh, "Calls", "other SB events")
    Rel(nbHandler, debounceRefresh, "Calls", "NB events")
    Rel(debounceRefresh, stateTimer, "Starts/Checks")
    Rel(stateTimer, refreshState, "Triggers", "After 500ms")
    Rel(immediateRefresh, refreshState, "Triggers", "Immediate")
    Rel(refreshState, ovn_state, "Updates")
    Rel(refreshState, scheduleReconcile, "Calls")
    Rel(scheduleReconcile, reconcileTimer, "Starts/Checks")
    Rel(reconcileTimer, onChange, "Triggers", "After 100ms")
    Rel(immediateRefresh, onChange, "Calls", "Direct")
```

### Key Code Structures

#### OVNState Structure

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

#### LocalRouterInfo Structure

```go
type LocalRouterInfo struct {
    RouterName  string   // NB Logical_Router name
    RouterUUID  string   // NB Logical_Router UUID
    LRPName     string   // NB Logical_Router_Port name
    LRPUUID     string   // NB Logical_Router_Port UUID
    LRPMAC      string   // NB Logical_Router_Port MAC
    LRPNetworks []string // NB Logical_Router_Port networks
    CRPort      string   // SB chassisredirect logical_port
}
```

#### RouteManager Structure

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

### Key Algorithms

#### State Refresh Algorithm

```
1. Query SB Port_Binding table
2. Query SB Chassis table
3. Build chassisHostname map
4. Find local chassisredirect ports
5. Query NB Logical_Router_Port table
6. Build LRP name → UUID map
7. Resolve local LRP UUIDs
8. Query NB Logical_Router table
9. Find routers with local LRPs
10. Build LocalRouterInfo list
11. Extract discovered networks from LRP networks
12. Query NB NAT table
13. Filter NAT entries to local routers
14. Extract SNAT IPs from SB gateway patch ports
15. Update OVNState atomically
```

#### Reconciliation Algorithm

```
1. Get OVN state snapshot
2. Compute effective network filters
3. Build hairpin MAC map
4. Ensure OVN NB default routes
5. Ensure OVN NB static MAC bindings
6. Ensure active priority lead
7. Ensure OVS MAC-tweak flows
8. Reconcile OVS hairpin flows
9. List current kernel routes
10. Add missing kernel routes
11. Remove stale kernel routes
12. List current FRR routes
13. Add missing FRR routes (batched)
14. Remove stale FRR routes (batched)
15. Reconcile veth leak networks
16. Reconcile FRR prefix-list
17. Check stale chassis
18. Verify routes installed
```

#### Event Debouncing Algorithm

```
On OVSDB event:
  IF chassisredirect change:
    Cancel pending state timer
    Trigger immediate state refresh
    Trigger direct reconcile
  ELSE:
    IF no state timer pending:
      Start 500ms state timer

On state timer expiry:
  Refresh state
  Start 100ms reconcile timer

On reconcile timer expiry:
  Trigger reconcile
```

---

## Deployment Architecture

### Deployment Diagram

```mermaid
C4Deployment
    title OVN Network Agent - Deployment

    Deployment_Node(compute1, "Compute Node 1", "Ubuntu 22.04") {
        Container(ovn_agent1, "OVN Network Agent", "Go binary", "Port: N/A")
        ContainerDb(ovn_sb1, "OVN SB DB (local)", "OVSDB", "Port: 6642")
        ContainerDb(ovn_nb1, "OVN NB DB (local)", "OVSDB", "Port: 6641")
        Container(frr1, "FRRouting", "BGP daemon", "Port: 179")
        Container(ovs1, "Open vSwitch", "ovs-vswitchd", "Port: 6653")
        ContainerDb(kernel1, "Linux Kernel", "Network stack")
        ContainerDb(nft1, "nftables", "Packet filter")
    }
    
    Deployment_Node(compute2, "Compute Node 2", "Ubuntu 22.04") {
        Container(ovn_agent2, "OVN Network Agent", "Go binary", "Port: N/A")
        ContainerDb(ovn_sb2, "OVN SB DB (local)", "OVSDB", "Port: 6642")
        ContainerDb(ovn_nb2, "OVN NB DB (local)", "OVSDB", "Port: 6641")
        Container(frr2, "FRRouting", "BGP daemon", "Port: 179")
        Container(ovs2, "Open vSwitch", "ovs-vswitchd", "Port: 6653")
        ContainerDb(kernel2, "Linux Kernel", "Network stack")
        ContainerDb(nft2, "nftables", "Packet filter")
    }
    
    Deployment_Node(controller, "Controller Node", "Ubuntu 22.04") {
        ContainerDb(ovn_nb_cluster, "OVN NB DB Cluster", "OVSDB", "Port: 6641")
        ContainerDb(ovn_sb_cluster, "OVN SB DB Cluster", "OVSDB", "Port: 6642")
    }
    
    Deployment_Node(router, "External Router", "Network Device") {
        Container(bgp_peer, "BGP Peer", "Router")
    }
    
    Rel(ovn_agent1, ovn_sb1, "Monitors", "OVSDB")
    Rel(ovn_agent1, ovn_nb1, "Monitors/Updates", "OVSDB")
    Rel(ovn_agent1, frr1, "Manages", "vtysh")
    Rel(ovn_agent1, ovs1, "Installs flows", "ovs-ofctl")
    Rel(ovn_agent1, kernel1, "Manages routes", "netlink")
    Rel(ovn_agent1, nft1, "Installs rules", "nft")
    Rel(frr1, bgp_peer, "BGP peering", "TCP 179")
    
    Rel(ovn_agent2, ovn_sb2, "Monitors", "OVSDB")
    Rel(ovn_agent2, ovn_nb2, "Monitors/Updates", "OVSDB")
    Rel(ovn_agent2, frr2, "Manages", "vtysh")
    Rel(ovn_agent2, ovs2, "Installs flows", "ovs-ofctl")
    Rel(ovn_agent2, kernel2, "Manages routes", "netlink")
    Rel(ovn_agent2, nft2, "Installs rules", "nft")
    Rel(frr2, bgp_peer, "BGP peering", "TCP 179")
    
    Rel(ovn_sb1, ovn_sb_cluster, "Replicates", "OVSDB")
    Rel(ovn_nb1, ovn_nb_cluster, "Replicates", "OVSDB")
    Rel(ovn_sb2, ovn_sb_cluster, "Replicates", "OVSDB")
    Rel(ovn_nb2, ovn_nb_cluster, "Replicates", "OVSDB")
```

### Deployment Notes

**Compute Nodes:**
- Run OVN Network Agent as systemd service
- Each agent monitors local OVN databases
- Agents coordinate via OVN databases (no direct communication)
- Gateway failover handled via OVN chassisredirect

**Controller Nodes:**
- Host OVN database clusters
- Provide high availability for OVN state
- Agents connect to cluster endpoints

**External Router:**
- BGP peer for route announcement
- Receives FIP/SNAT routes from FRR
- Provides external connectivity

---

## Data Flow

### Data Flow Diagram

```mermaid
graph LR
    subgraph "OVN Databases"
        SB[OVN SB DB]
        NB[OVN NB DB]
    end
    
    subgraph "OVN Network Agent"
        OVN[OVN Client]
        AGT[Agent]
        RM[Route Manager]
    end
    
    subgraph "System"
        KNL[Kernel Routes]
        FRR[FRR Routes]
        OVS[OVS Flows]
        NFT[nftables]
    end
    
    SB -->|Monitor| OVN
    NB -->|Monitor/Update| OVN
    OVN -->|State| AGT
    AGT -->|Commands| RM
    RM -->|Manage| KNL
    RM -->|Manage| FRR
    RM -->|Manage| OVS
    RM -->|Manage| NFT
```

### Data Flow Description

1. **Monitoring Flow**: OVN Client monitors SB and NB databases for changes
2. **State Flow**: OVN Client builds derived state and provides to Agent
3. **Command Flow**: Agent issues commands to Route Manager based on state
4. **Management Flow**: Route Manager manages routes, flows, and rules in system

---

## Security Considerations

### Security Diagram

```mermaid
C4Container
    title Security Considerations

    Container(ovn_agent, "OVN Network Agent", "Runs as root")
    ContainerDb(ovn_sb, "OVN SB DB", "Requires authentication")
    ContainerDb(ovn_nb, "OVN NB DB", "Requires authentication")
    ContainerDb(kernel, "Linux Kernel", "CAP_NET_ADMIN required")
    ContainerDb(frr, "FRRouting", "vtysh access control")
    ContainerDb(nftables, "nftables", "Root access required")
    
    Rel(ovn_agent, ovn_sb, "Connects", "TLS/SSL recommended")
    Rel(ovn_agent, ovn_nb, "Connects", "TLS/SSL recommended")
    Rel(ovn_agent, kernel, "Modifies", "Requires root")
    Rel(ovn_agent, frr, "Modifies", "vtysh access control")
    Rel(ovn_agent, nftables, "Modifies", "Requires root")
```

### Security Notes

1. **Privilege Requirements**: Must run as root (CAP_NET_ADMIN)
2. **Network Exposure**: Listens on OVSDB connections
3. **Configuration**: Sensitive data in config file (DB endpoints)
4. **nftables**: Rules affect system-wide packet processing
5. **VRF Access**: Can modify VRF routing tables
6. **TLS/SSL**: Recommended for OVSDB connections

---

## Performance Characteristics

### Performance Diagram

```mermaid
graph TB
    subgraph "Scalability"
        R1[Route Count: 1000s]
        B1[Batch Size: 500]
        D1[Debounce: 500ms]
    end
    
    subgraph "Resource Usage"
        M1[Memory: Proportional to routes]
        C1[CPU: Event-driven, low idle]
        N1[Network: OVSDB monitoring only]
    end
    
    subgraph "Latency"
        F1[Failover: <1s]
        R2[Reconciliation: <5s]
        E1[Event processing: <100ms]
    end
```

### Performance Notes

- **Scalability**: Tested with thousands of FIPs
- **Batch Operations**: FRR routes batched (500 per call)
- **Debouncing**: Coalesces rapid changes
- **Efficient Queries**: OVSDB monitoring with incremental updates
- **Memory**: Proportional to route count
- **CPU**: Event-driven, low idle usage
- **Network**: OVSDB monitoring traffic only
- **Failover**: <1s for chassisredirect changes
- **Reconciliation**: <5s for full cycle
- **Event Processing**: <100ms per event

---

## Monitoring and Observability

### Monitoring Diagram

```mermaid
C4Container
    title Monitoring and Observability

    Container(ovn_agent, "OVN Network Agent", "Structured logging")
    ContainerDb(logs, "System Logs", "journald/syslog")
    ContainerDb(metrics, "Metrics", "Future enhancement")
    ContainerDb(traces, "Traces", "Future enhancement")
    
    Rel(ovn_agent, logs, "Writes", "slog structured logs")
    Rel(ovn_agent, metrics, "Exports", "Prometheus format")
    Rel(ovn_agent, traces, "Exports", "OpenTelemetry")
```

### Observability Notes

**Current:**
- Structured logging with slog
- Log levels: debug, info, warn, error
- Dry-run mode for testing

**Future Enhancements:**
- Prometheus metrics
- OpenTelemetry tracing
- Health check endpoints
- Performance profiling

---

## Troubleshooting Guide

### Troubleshooting Flow

```mermaid
graph TD
    A[Issue Detected] --> B{Check logs}
    B --> C{Dry-run mode?}
    C -->|Yes| D[Disable dry-run]
    C -->|No| E{OVN connection?}
    E -->|Failed| F[Check DB endpoints]
    E -->|OK| G{Bridge device?}
    G -->|Missing| H[Create br-ex]
    G -->|OK| I{Permissions?}
    I -->|Denied| J[Run as root]
    I -->|OK| K{Routes appearing?}
    K -->|No| L[Check FRR connectivity]
    K -->|Yes| M[Verify OVS flows]
    M --> N[Check nftables rules]
```

### Common Issues

1. **Routes not appearing**: Check `dry_run` mode
2. **OVN connection failed**: Verify DB endpoints
3. **Bridge device missing**: Ensure `br-ex` exists
4. **Permission denied**: Run as root
5. **FRR routes not syncing**: Check vtysh connectivity

### Debug Commands

```bash
# Enable debug logging
ovn-network-agent --log-level debug

# Dry run mode
ovn-network-agent --dry-run

# Check systemd status
systemctl status ovn-network-agent

# View logs
journalctl -u ovn-network-agent -f
```
