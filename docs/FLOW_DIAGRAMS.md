# OVN Network Agent - Flow Diagrams

## Table of Contents
1. [Startup Flow](#startup-flow)
2. [Reconciliation Flow](#reconciliation-flow)
3. [Event Handling Flow](#event-handling-flow)
4. [Route Synchronization Flow](#route-synchronization-flow)
5. [Gateway Failover Flow](#gateway-failover-flow)
6. [Port Forwarding Flow](#port-forwarding-flow)
7. [Graceful Shutdown Flow](#graceful-shutdown-flow)
8. [Stale Chassis Cleanup Flow](#stale-chassis-cleanup-flow)

---

## Startup Flow

```mermaid
sequenceDiagram
    participant Main as main()
    participant Config as loadConfig()
    participant Agent as Agent
    participant Routing as RouteManager
    participant OVN as OVNClient
    participant Kernel as Kernel
    participant OVS as OVS

    Main->>Config: loadConfig(args)
    Config-->>Main: Config

    Main->>Main: setupLogging()

    Main->>Agent: NewAgent(cfg)
    Agent->>Routing: NewRouteManager(cfg)
    Routing-->>Agent: RouteManager
    Agent->>OVN: NewOVNClient(cfg, onChange)
    OVN-->>Agent: OVNClient
    Agent-->>Main: Agent

    Main->>Agent: Run(ctx)

    Agent->>Kernel: CheckBridgeDevice()
    Kernel-->>Agent: OK

    Agent->>Kernel: EnsureBridgeIP(bridgeIP)
    Kernel-->>Agent: OK

    Agent->>Kernel: EnableProxyARP()
    Kernel-->>Agent: OK

    Agent->>Routing: SetupVethLeak()
    Routing->>Kernel: Create veth pair
    Routing->>Kernel: Configure VRF
    Routing->>Kernel: Add routes
    Routing-->>Agent: OK

    Agent->>Routing: SetupPortForward()
    Routing->>Kernel: Configure nftables
    Routing-->>Agent: OK

    loop Connect to OVN
        Agent->>OVN: Connect(ctx)
        OVN->>OVN: Connect to SB DB
        OVN->>OVN: Monitor SB tables
        OVN->>OVN: Connect to NB DB
        OVN->>OVN: Monitor NB tables
        OVN->>OVN: refreshState()
        OVN-->>Agent: OK
    end

    Agent->>OVN: RestoreDrainedGateways()
    OVN->>OVN: Update priorities
    OVN-->>Agent: OK

    Agent->>Agent: reconcile()

    Agent->>Agent: Event Loop
    Note over Agent: Wait for events or periodic tick
```

---

## Reconciliation Flow

```mermaid
sequenceDiagram
    participant Agent as Agent
    participant OVN as OVNClient
    participant State as OVNState
    participant Routing as RouteManager
    participant Kernel as Kernel
    participant FRR as FRR
    participant OVS as OVS

    Agent->>OVN: GetState()
    OVN->>State: Lock (R)
    State-->>OVN: State snapshot
    OVN-->>Agent: OVNState

    Agent->>Agent: computeEffectiveNetworks()

    Agent->>Agent: Build hairpinMACMap()

    Agent->>OVN: EnsureGatewayRouting()
    OVN->>OVN: For each local router
    OVN->>OVN: virtualGatewayIP()
    OVN->>OVN: ensureDefaultRoute()
    OVN->>OVN: ensureStaticMACBinding()
    OVN-->>Agent: OK

    Agent->>OVN: EnsureActivePriorityLead()
    OVN->>OVN: List Gateway_Chassis
    OVN->>OVN: Compare priorities
    OVN->>OVN: Update if needed
    OVN-->>Agent: OK

    Agent->>Routing: EnsureOVSFlows()
    Routing->>OVS: discoverPatchPort()
    OVS-->>Routing: patch port
    Routing->>OVS: getOFPort()
    OVS-->>Routing: ofport
    Routing->>OVS: GetBridgeMAC()
    OVS-->>Routing: MAC
    Routing->>OVS: Add MAC-tweak flows
    OVS-->>Routing: OK
    Routing-->>Agent: OK

    Agent->>Routing: ReconcileOVSHairpinFlows()
    Routing->>OVS: Delete old hairpin flows
    Routing->>OVS: Add new hairpin flows
    OVS-->>Routing: OK
    Routing-->>Agent: OK

    Agent->>Routing: ListKernelRoutes()
    Routing->>Kernel: Query routes
    Kernel-->>Routing: Current routes
    Routing-->>Agent: Route list

    Agent->>Agent: Compare desired vs actual

    loop Add missing kernel routes
        Agent->>Routing: AddKernelRoute(ip)
        Routing->>Kernel: Add /32 route
        Kernel-->>Routing: OK
        Routing-->>Agent: OK
    end

    loop Remove stale kernel routes
        Agent->>Routing: DelKernelRoute(ip)
        Routing->>Kernel: Delete /32 route
        Kernel-->>Routing: OK
        Routing-->>Agent: OK
    end

    Agent->>Routing: ListFRRRoutes()
    Routing->>FRR: vtysh show ip route
    FRR-->>Routing: Route list
    Routing-->>Agent: Route list

    loop Add missing FRR routes (batched)
        Agent->>Routing: AddFRRRoutes(ips)
        Routing->>FRR: vtysh batch add
        FRR-->>Routing: OK
        Routing-->>Agent: OK
    end

    loop Remove stale FRR routes (batched)
        Agent->>Routing: DelFRRRoutes(ips)
        Routing->>FRR: vtysh batch del
        FRR-->>Routing: OK
        Routing-->>Agent: OK
    end

    Agent->>Routing: ReconcileVethLeakNetworks()
    Routing->>Kernel: List current leak routes
    Kernel-->>Routing: Route list
    Routing->>Agent: Current networks

    loop Add missing leak routes
        Agent->>Routing: Add leak route
        Routing->>Kernel: Add route + rule
        Kernel-->>Routing: OK
    end

    loop Remove stale leak routes
        Agent->>Routing: Remove leak route
        Routing->>Kernel: Del route + rule
        Kernel-->>Routing: OK
    end

    Agent->>Routing: ReconcileFRRPrefixList()
    Routing->>FRR: Update prefix-list
    FRR-->>Routing: OK

    Agent->>Agent: Check stale chassis
    Agent->>OVN: CleanupStaleChassisEntries()
    OVN->>OVN: Remove old entries
    OVN-->>Agent: OK

    Agent->>Agent: Post-change verification
```

---

## Event Handling Flow

```mermaid
sequenceDiagram
    participant OVSDB as OVSDB
    participant SBHandler as sbEventHandler
    participant NBHandler as nbEventHandler
    participant OVN as OVNClient
    participant Timer as Debounce Timer
    participant State as OVNState
    participant Agent as Agent

    OVSDB->>SBHandler: OnAdd/OnUpdate/OnDelete
    SBHandler->>SBHandler: isChassisRedirect()?

    alt Chassisredirect change
        SBHandler->>OVN: immediateStateRefresh()
        OVN->>OVN: Cancel pending timer
        OVN->>OVN: refreshState() (async)
        OVN->>State: Lock (W)
        OVN->>OVSDB: List SB Port_Binding
        OVN->>OVSDB: List SB Chassis
        OVN->>OVSDB: List NB LRP
        OVN->>OVSDB: List NB Router
        OVN->>OVSDB: List NB NAT
        OVN->>State: Update state
        OVN->>State: Unlock
        OVN->>Agent: onChange() (direct)
        Agent->>Agent: reconcile()
    else Other SB change
        SBHandler->>OVN: debounceStateRefresh()
        OVN->>Timer: Timer already pending?
        alt No timer pending
            OVN->>Timer: Start 500ms timer
        end
    end

    OVSDB->>NBHandler: OnAdd/OnUpdate/OnDelete
    NBHandler->>OVN: debounceStateRefresh()
    OVN->>Timer: Timer already pending?
    alt No timer pending
        OVN->>Timer: Start 500ms timer
    end

    Timer->>OVN: Timer expired
    OVN->>Timer: Clear timer
    OVN->>OVN: refreshState()
    OVN->>State: Lock (W)
    OVN->>OVSDB: Query all tables
    OVN->>State: Update state
    OVN->>State: Unlock
    OVN->>OVN: scheduleReconcile()
    OVN->>Timer: Start 100ms timer

    Timer->>OVN: Timer expired
    OVN->>Timer: Clear timer
    OVN->>Agent: onChange()
    Agent->>Agent: triggerReconcile()
    Agent->>Agent: reconcile()
```

---

## Route Synchronization Flow

```mermaid
sequenceDiagram
    participant Agent as Agent
    participant Routing as RouteManager
    participant Kernel as Kernel
    participant FRR as FRR
    participant Veth as Veth Leak

    Note over Agent: Reconciliation starts

    Agent->>Routing: ListKernelRoutes()
    Routing->>Kernel: netlink.RouteList()
    Kernel-->>Routing: Current /32 routes
    Routing-->>Agent: IP list

    Agent->>Agent: Compute desired IPs
    Note over Agent: desired = FIPs + SNATIPs

    Agent->>Agent: Compare current vs desired

    par Add missing routes
        loop For each missing IP
            Agent->>Routing: AddKernelRoute(ip)
            Routing->>Kernel: netlink.RouteReplace()
            alt Using custom table
                Routing->>Kernel: netlink.RuleAdd()
            end
            Kernel-->>Routing: OK
            Routing-->>Agent: OK
        end
    and Remove stale routes
        loop For each stale IP
            Agent->>Routing: DelKernelRoute(ip)
            alt Using custom table
                Routing->>Kernel: netlink.RuleDel()
            end
            Routing->>Kernel: netlink.RouteDel()
            Kernel-->>Routing: OK
            Routing-->>Agent: OK
        end
    end

    Agent->>Routing: ListFRRRoutes()
    Routing->>FRR: vtysh show ip route vrf static
    FRR-->>Routing: Route list
    Routing-->>Agent: IP list

    par Add missing FRR routes
        loop Batch of 500 IPs
            Agent->>Routing: AddFRRRoutes(batch)
            Routing->>FRR: vtysh -c "conf t" -c "vrf vrf-provider"
            Routing->>FRR: vtysh -c "ip route X/32 nexthop"
            Routing->>FRR: vtysh -c "exit-vrf" -c "end"
            FRR-->>Routing: OK
            Routing-->>Agent: OK
        end
    and Remove stale FRR routes
        loop Batch of 500 IPs
            Agent->>Routing: DelFRRRoutes(batch)
            Routing->>FRR: vtysh -c "conf t" -c "vrf vrf-provider"
            Routing->>FRR: vtysh -c "no ip route X/32 nexthop"
            Routing->>FRR: vtysh -c "exit-vrf" -c "end"
            FRR-->>Routing: OK
            Routing-->>Agent: OK
        end
    end

    Agent->>Routing: ReconcileVethLeakNetworks()
    Routing->>Veth: List current leak routes
    Veth-->>Routing: Network list

    par Add missing leak routes
        loop For each missing network
            Agent->>Routing: Add leak route
            Routing->>Kernel: Add route in VRF
            Routing->>Kernel: Add policy rule
            Kernel-->>Routing: OK
        end
    and Remove stale leak routes
        loop For each stale network
            Agent->>Routing: Remove leak route
            Routing->>Kernel: Del policy rule
            Routing->>Kernel: Del route
            Kernel-->>Routing: OK
        end
    end

    Agent->>Agent: Verify routes installed
    Note over Agent: Post-change verification
```

---

## Gateway Failover Flow

```mermaid
sequenceDiagram
    participant Node1 as Node1 (Active)
    participant Node2 as Node2 (Standby)
    participant OVN_SB as OVN SB DB
    participant OVN_NB as OVN NB DB
    participant Agent1 as Agent on Node1
    participant Agent2 as Agent on Node2

    Note over Node1: Active chassis (priority 2)
    Note over Node2: Standby chassis (priority 1)

    Node1->>Node1: Failure / Shutdown

    Agent1->>Agent1: drainOnShutdown?
    alt Drain enabled
        Agent1->>OVN_NB: DrainGateways()
        Agent1->>OVN_NB: Set priority to 0
        OVN_NB-->>Agent1: OK
        Note over Agent1: Wait for migration
    end

    OVN_SB->>OVN_SB: Detect chassis missing
    OVN_SB->>OVN_SB: Update Port_Binding.chassis

    OVN_SB->>Agent2: SB event (chassisredirect change)
    Agent2->>Agent2: immediateStateRefresh()

    Agent2->>OVN_SB: List Port_Binding
    Agent2->>OVN_SB: List Chassis
    OVN_SB-->>Agent2: Data

    Agent2->>Agent2: Detect new local router
    Note over Agent2: chassisredirect now on Node2

    Agent2->>Agent2: reconcile()

    Agent2->>OVN_NB: EnsureGatewayRouting()
    Agent2->>OVN_NB: Add default route
    Agent2->>OVN_NB: Add static MAC binding
    OVN_NB-->>Agent2: OK

    Agent2->>Agent2: Add kernel routes
    Agent2->>Agent2: Add FRR routes
    Agent2->>Agent2: Install OVS flows

    Note over Node2: Now active (priority 2)

    Agent1->>Agent1: Cleanup on shutdown
    Agent1->>Agent1: Remove all routes
    Agent1->>Agent1: Remove OVS flows
    Agent1->>Agent1: Exit

    Note over Node2: Traffic flowing through Node2
```

---

## Port Forwarding Flow

```mermaid
sequenceDiagram
    participant Client as External Client
    participant VIP as VIP (anycast)
    participant NFT as nftables
    participant CT as Conntrack
    participant Backend as Backend Server
    participant VRF as VRF Routing
    participant Return as Return Path

    Client->>VIP: TCP SYN to VIP:80
    VIP->>NFT: prerouting_ctzone
    NFT->>CT: ct zone set 64000
    CT-->>NFT: OK
    NFT->>NFT: prerouting_dnat

    alt Single backend
        NFT->>NFT: dnat to backend:8080
    else Multiple backends
        NFT->>NFT: jhash(src_ip) mod N
        NFT->>NFT: dnat to backend_N:8080
    end

    NFT->>NFT: prerouting_fwmark
    NFT->>NFT: ct status dnat?
    alt DNAT'd packet
        NFT->>NFT: mark 0x100
    else Reply packet
        NFT->>NFT: mark 0x200
    end

    NFT->>VRF: Policy routing
    alt Mark 0x100
        VRF->>VRF: lookup main table
    else Mark 0x200
        VRF->>VRF: lookup VRF table
    end

    alt Backend on remote host
        VRF->>Backend: Forward packet
        Backend-->>VRF: Process request
        Backend->>Return: Send reply
        Return->>NFT: output_fwmark
        NFT->>NFT: mark 0x200
        NFT->>NFT: postrouting_snat
        alt Masquerade enabled
            NFT->>NFT: SNAT with VIP
        end
        NFT->>NFT: postrouting_fwmark_clear
        NFT->>NFT: clear 0x200
        NFT->>Client: Send reply
    else Backend on same host
        VRF->>Backend: Local delivery
        Backend-->>VRF: Process request
        Backend->>NFT: output_ctzone
        NFT->>CT: ct zone set 64000
        NFT->>NFT: output_fwmark
        NFT->>NFT: mark 0x200
        NFT->>NFT: postrouting_snat
        alt Masquerade enabled
            NFT->>NFT: SNAT with VIP
        end
        NFT->>NFT: forward_veth_guard
        NFT->>NFT: Validate return path
        NFT->>Client: Send reply
    end
```

---

## Graceful Shutdown Flow

```mermaid
sequenceDiagram
    participant Signal as OS Signal
    participant Agent as Agent
    participant OVN as OVNClient
    participant Routing as RouteManager
    participant Kernel as Kernel
    participant FRR as FRR
    participant OVS as OVS

    Signal->>Agent: SIGINT/SIGTERM
    Agent->>Agent: Cancel context

    Agent->>Agent: drainOnShutdown?
    alt Drain enabled
        Agent->>OVN: DrainGateways()
        OVN->>OVN: For each local router
        OVN->>OVN: Set Gateway_Chassis priority to 0
        OVN->>OVN: Wait for migration
        Note over OVN: ovn-northd moves to standby
        OVN-->>Agent: OK
    end

    Agent->>Agent: cleanupOnShutdown?
    alt Cleanup enabled
        Agent->>Routing: CleanupRoutingTable()
        Routing->>Kernel: Remove all routes
        Routing->>Kernel: Remove all rules
        Kernel-->>Routing: OK

        Agent->>Routing: RemoveOVSFlows()
        Routing->>OVS: Delete MAC-tweak flows
        Routing->>OVS: Delete hairpin flows
        OVS-->>Routing: OK

        Agent->>Routing: TeardownVethLeak()
        Routing->>Kernel: Remove veth pair
        Routing->>Kernel: Remove leak routes
        Routing->>Kernel: Remove policy rules
        Kernel-->>Routing: OK

        Agent->>Routing: TeardownPortForward()
        Routing->>Kernel: Remove nftables rules
        Kernel-->>Routing: OK

        Agent->>Routing: RemoveBridgeIP()
        Routing->>Kernel: Delete bridge IP
        Kernel-->>Routing: OK

        Agent->>FRR: Remove all FRR routes
        FRR-->>Agent: OK
    end

    Agent->>OVN: Close()
    OVN->>OVN: Cancel timers
    OVN->>OVN: Close SB client
    OVN->>OVN: Close NB client

    Agent->>Agent: Exit
```

---

## Stale Chassis Cleanup Flow

```mermaid
sequenceDiagram
    participant Agent as Agent
    participant OVN as OVNClient
    participant State as OVNState
    participant Timer as Cleanup Timer
    participant OVN_NB as OVN NB DB

    Note over Agent: Periodic reconciliation

    Agent->>OVN: GetState()
    OVN->>State: Lock (R)
    State-->>OVN: State snapshot
    OVN-->>Agent: OVNState

    Agent->>Agent: Check missingChassis map

    loop For each missing chassis
        Agent->>Agent: chassis in AllChassisNames?

        alt Chassis still missing
            Agent->>Agent: time elapsed > grace period?
            alt Grace period expired
                Agent->>OVN: CleanupStaleChassisEntries()
                OVN->>OVN_NB: List Gateway_Chassis
                OVN->>OVN_NB: List Static_MAC_Binding
                OVN->>OVN_NB: List Logical_Router_Static_Route
                OVN_NB-->>OVN: Entries

                loop For each stale entry
                    OVN->>OVN_NB: Delete entry
                    OVN_NB-->>OVN: OK
                end

                OVN-->>Agent: OK
                Agent->>Agent: Remove from missingChassis
            else Grace period not expired
                Note over Agent: Wait for next cycle
            end
        else Chassis reappeared
            Agent->>Agent: Remove from missingChassis
            Note over Agent: Chassis recovered
        end
    end

    Agent->>Agent: Check AllChassisNames

    loop For each chassis in state
        Agent->>Agent: chassis in missingChassis?
        alt Not in missingChassis
            Note over Agent: Chassis present
        else In missingChassis
            Note over Agent: Already tracking
        end
    end

    Note over Agent: Detect new missing chassis
    Agent->>Agent: Compare with previous AllChassisNames

    loop For each chassis that disappeared
        Agent->>Agent: Add to missingChassis
        Agent->>Agent: Record timestamp
        Note over Agent: Add random jitter (0-30s)
    end
```

---

## Method Call Sequence - State Refresh

```mermaid
sequenceDiagram
    participant OVN as OVNClient
    participant SB as SB Client
    participant NB as NB Client
    participant State as OVNState

    OVN->>SB: List(ctx, &PortBinding)
    SB-->>OVN: portBindings

    OVN->>SB: List(ctx, &Chassis)
    SB-->>OVN: chassis

    OVN->>OVN: Build chassisHostname map
    OVN->>OVN: Build allChassisNames set

    OVN->>OVN: Find local chassisredirect ports
    loop For each portBinding
        OVN->>OVN: Check type == "chassisredirect"
        OVN->>OVN: Check chassis == local
        OVN->>OVN: Extract LRP name
        OVN->>OVN: Add to localLRPNames
    end

    OVN->>NB: List(ctx, &LogicalRouterPort)
    NB-->>OVN: lrps

    OVN->>OVN: Build lrpNameToUUID map
    OVN->>OVN: Build lrpNameToMAC map

    OVN->>OVN: Resolve local LRP UUIDs
    loop For each localLRPName
        OVN->>OVN: Lookup UUID in lrpNameToUUID
        OVN->>OVN: Add to localLRPUUIDs
    end

    OVN->>NB: List(ctx, &LogicalRouter)
    NB-->>OVN: routers

    OVN->>OVN: Find routers with local LRPs
    loop For each router
        OVN->>OVN: Check if router.Ports contains localLRPUUID
        alt Has local LRP
            OVN->>OVN: Create LocalRouterInfo
            OVN->>OVN: Build natUUIDToRouterMAC
            OVN->>OVN: Add to localRouters
        end
    end

    OVN->>OVN: Extract discovered networks
    loop For each localRouter
        loop For each LRPNetwork
            OVN->>OVN: Parse CIDR
            OVN->>OVN: Add to discoveredNets
        end
    end

    OVN->>OVN: uniqueNetworks(discoveredNets)

    OVN->>NB: List(ctx, &NAT)
    NB-->>OVN: nats

    OVN->>OVN: Filter NAT entries
    loop For each nat
        OVN->>OVN: Check natUUID in natUUIDToRouterMAC
        OVN->>OVN: Check IP in effectiveFilters
        OVN->>OVN: Add to fips or snatIPs
        OVN->>OVN: Add to natIPToRouterMAC
    end

    OVN->>OVN: Extract SNAT IPs from SB
    loop For each portBinding
        OVN->>OVN: Check type == "patch"
        OVN->>OVN: Check peer in localLRPNames
        OVN->>OVN: Check device_owner == gateway
        loop For each natAddress
            OVN->>OVN: parseNatAddressIPs()
            OVN->>OVN: Add to snatIPs
            OVN->>OVN: Add to natIPToRouterMAC
        end
    end

    OVN->>State: Lock (W)
    OVN->>State: Update all fields
    OVN->>State: Unlock
```

---

## Method Call Sequence - OVS Flow Management

```mermaid
sequenceDiagram
    participant Agent as Agent
    participant Routing as RouteManager
    participant OVS as OVS

    Agent->>Routing: EnsureOVSFlows()

    alt First call (no cache)
        Routing->>OVS: discoverPatchPort()
        OVS->>OVS: ovs-vsctl list-ports br-ex
        OVS-->>Routing: port list
        loop For each port
            Routing->>OVS: ovs-vsctl get Interface type
            OVS-->>Routing: type
            alt type == "patch"
                Routing->>Routing: Found patch port
            end
        end
        OVS-->>Routing: patchPort

        Routing->>OVS: getOFPort(patchPort)
        OVS->>OVS: ovs-vsctl get Interface ofport
        OVS-->>Routing: ofport

        Routing->>OVS: GetBridgeMAC()
        OVS->>OVS: netlink.LinkByName()
        OVS-->>Routing: MAC address

        Routing->>Routing: Cache values
    end

    Routing->>OVS: ovs-ofctl del-flows cookie=0x999/-1
    OVS-->>Routing: OK

    Routing->>Routing: MACTweakFlow(0x999, ofport, mac, false)
    Note over Routing: cookie=0x999,priority=900,ip,in_port=X,<br/>actions=mod_dl_dst:MAC,NORMAL
    Routing->>OVS: ovs-ofctl add-flow [IPv4 flow]
    OVS-->>Routing: OK

    Routing->>Routing: MACTweakFlow(0x999, ofport, mac, true)
    Note over Routing: cookie=0x999,priority=900,ipv6,in_port=X,<br/>actions=mod_dl_dst:MAC,NORMAL
    Routing->>OVS: ovs-ofctl add-flow [IPv6 flow]
    OVS-->>Routing: OK

    Routing-->>Agent: OK

    Agent->>Routing: ReconcileOVSHairpinFlows(ipToRouterMAC)

    Routing->>OVS: ovs-ofctl del-flows cookie=0x998/-1
    OVS-->>Routing: OK

    loop For each IP in ipToRouterMAC
        Routing->>Routing: HairpinFlow(0x998, ofport, ip, bridgeMAC, routerMAC, ipv4)
        Note over Routing: cookie=0x998,priority=910,ip,in_port=X,<br/>ip_dst=IP/32,actions=mod_dl_src:bridgeMAC,<br/>mod_dl_dst:routerMAC,output:in_port
        Routing->>OVS: ovs-ofctl add-flow [hairpin flow]
        OVS-->>Routing: OK
    end

    Routing-->>Agent: OK
```

---

## Method Call Sequence - Veth Leak Setup

```mermaid
sequenceDiagram
    participant Agent as Agent
    participant Routing as RouteManager
    participant Kernel as Kernel

    Agent->>Routing: SetupVethLeak()

    Routing->>Kernel: netlink.LinkByName(veth-default)
    alt Link exists
        Kernel-->>Routing: veth-default
        Note over Routing: Reuse existing veth pair
    else Link doesn't exist
        Routing->>Kernel: netlink.LinkAdd(veth pair)
        Kernel-->>Routing: OK
        Routing->>Kernel: netlink.LinkByName(veth-default)
        Kernel-->>Routing: veth-default
    end

    Routing->>Kernel: netlink.LinkByName(veth-provider)
    Kernel-->>Routing: veth-provider

    Routing->>Kernel: netlink.LinkByName(vrf-provider)
    Kernel-->>Routing: VRF link

    Routing->>Kernel: netlink.LinkSetMaster(veth-provider, vrf-provider)
    Kernel-->>Routing: OK

    Routing->>Kernel: netlink.AddrReplace(veth-default, nexthop/30)
    Kernel-->>Routing: OK

    Routing->>Kernel: netlink.AddrReplace(veth-provider, providerIP/30)
    Kernel-->>Routing: OK

    Routing->>Kernel: netlink.LinkSetUp(veth-default)
    Kernel-->>Routing: OK

    Routing->>Kernel: netlink.LinkSetUp(veth-provider)
    Kernel-->>Routing: OK

    Routing->>Kernel: netlink.LinkByName(veth-default)
    Kernel-->>Routing: veth-default (refreshed)

    Routing->>Kernel: netlink.LinkByName(veth-provider)
    Kernel-->>Routing: veth-provider (refreshed)

    Routing->>Kernel: netlink.NeighSet(veth-default, providerIP, providerMAC, NUD_PERMANENT)
    Kernel-->>Routing: OK

    Routing->>Kernel: netlink.NeighSet(veth-provider, nexthop, defaultMAC, NUD_PERMANENT)
    Kernel-->>Routing: OK

    Routing->>Kernel: netlink.RouteReplace(default via providerIP dev veth-default table leak_table)
    Kernel-->>Routing: OK

    alt Static network_cidr configured
        Routing->>Routing: ReconcileVethLeakNetworks(networkFilters)
        Note over Routing: Add per-network routes and rules
    end

    Routing-->>Agent: OK
```
