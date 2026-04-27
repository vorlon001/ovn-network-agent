# OVN Network Agent - Documentation

Welcome to the comprehensive documentation for the OVN Network Agent. This documentation provides in-depth information about the architecture, design, and implementation of the agent.

## Overview

The OVN Network Agent is an event-driven daemon that monitors OVN (Open Virtual Network) databases and synchronizes network state between OVN and the local system. It's designed for OpenStack environments using OVN for networking.

## Documentation Structure

### [Architecture Documentation](./ARCHITECTURE.md)
Comprehensive architecture documentation covering:
- System architecture and component design
- Data flow and state management
- Event handling and reconciliation
- Routing architecture
- Port forwarding
- High availability
- Configuration
- Dependencies
- Security considerations
- Performance characteristics
- Troubleshooting

**Best for:** Understanding the overall system design and how components interact.

### [Flow Diagrams](./FLOW_DIAGRAMS.md)
Detailed sequence diagrams covering:
- Startup flow
- Reconciliation flow
- Event handling flow
- Route synchronization flow
- Gateway failover flow
- Port forwarding flow
- Graceful shutdown flow
- Stale chassis cleanup flow
- Method call sequences

**Best for:** Understanding the execution flow and method call order.

### [C4 Architecture](./C4_ARCHITECTURE.md)
C4 model documentation covering:
- Level 1: System Context
- Level 2: Container View
- Level 3: Component View
- Level 4: Code View
- Deployment architecture
- Data flow
- Security considerations
- Performance characteristics
- Monitoring and observability
- Troubleshooting guide

**Best for:** Understanding the system at different levels of abstraction.

### [API Reference](./API_REFERENCE.md)
Complete API reference covering:
- Main package functions
- Agent package types and methods
- OVN package types and methods
- Routing package types and methods
- OVS package functions
- nftables package functions
- Config package types
- Helper functions
- Constants
- Error handling
- Usage examples
- Best practices
- Performance considerations
- Security considerations

**Best for:** Understanding the code API and implementation details.

## Quick Start

### For New Contributors

1. Start with [Architecture Documentation](./ARCHITECTURE.md) to understand the system
2. Review [Flow Diagrams](./FLOW_DIAGRAMS.md) to understand execution flow
3. Explore [C4 Architecture](./C4_ARCHITECTURE.md) for component relationships
4. Reference [API Reference](./API_REFERENCE.md) for implementation details

### For Operators

1. Read [Architecture Documentation](./ARCHITECTURE.md) sections:
   - Overview
   - Configuration
   - High Availability
   - Troubleshooting
2. Review [C4 Architecture](./C4_ARCHITECTURE.md) sections:
   - Deployment Architecture
   - Security Considerations
   - Monitoring and Observability

### For Developers

1. Study [Architecture Documentation](./ARCHITECTURE.md) for design patterns
2. Analyze [Flow Diagrams](./FLOW_DIAGRAMS.md) for algorithm understanding
3. Use [C4 Architecture](./C4_ARCHITECTURE.md) for component design
4. Reference [API Reference](./API_REFERENCE.md) for code implementation

## Key Concepts

### Event-Driven Architecture
The agent is event-driven, reacting to OVSDB changes in real-time. It monitors both Southbound (SB) and Northbound (NB) databases and performs targeted writes to both OVN NB and the local system.

### Reconciliation
The agent performs periodic reconciliation (default: every 60s) as a safety net to ensure the local state matches the desired state from OVN.

### Debouncing
Rapid changes are coalesced using debouncing:
- State refresh: 500ms after first event
- Reconciliation: 100ms after state refresh
- Chassisredirect changes: Bypass debounce for fast failover

### Idempotency
All operations are idempotent and safe to run multiple times. This ensures reliability and simplifies error recovery.

### High Availability
The agent supports gateway chassis failover with:
- Fast failover detection (<1s)
- Graceful shutdown with drain mode
- Stale chassis cleanup
- Priority-based active/standby

## Component Overview

### Main Package
Entry point and lifecycle management. Handles configuration loading, logging setup, and signal handling.

### Agent Package
Main orchestrator that coordinates all components. Runs the event loop and performs reconciliation.

### OVN Package
OVSDB client wrapper that manages connections, monitors databases, and performs state refreshes.

### Routing Package
Manages kernel routes, FRR routes, OVS flows, veth leak, and port forwarding.

### OVS Package
Handles OpenFlow flow installation and removal on the provider bridge.

### nftables Package
Generates and manages nftables rules for port forwarding (DNAT).

### Config Package
Handles configuration loading from multiple sources with priority: CLI flags > env vars > config file > defaults.

## Data Flow

```
OVN Databases (SB/NB)
    ↓
OVN Client (monitoring)
    ↓
State Cache (derived state)
    ↓
Agent (reconciliation)
    ↓
Route Manager (operations)
    ↓
System (kernel, FRR, OVS, nftables)
```

## Key Algorithms

### State Refresh
1. Query SB Port_Binding and Chassis tables
2. Query NB Logical_Router_Port, Logical_Router, NAT tables
3. Build chassis hostname map
4. Find local chassisredirect ports
5. Resolve local LRP UUIDs
6. Find routers with local LRPs
7. Extract discovered networks
8. Filter NAT entries
9. Extract SNAT IPs from SB gateway ports
10. Update state atomically

### Reconciliation
1. Get OVN state snapshot
2. Compute effective network filters
3. Build hairpin MAC map
4. Ensure OVN NB default routes
5. Ensure OVN NB static MAC bindings
6. Ensure active priority lead
7. Ensure OVS flows
8. Reconcile OVS hairpin flows
9. Reconcile kernel routes
10. Reconcile FRR routes
11. Reconcile veth leak networks
12. Reconcile FRR prefix-list
13. Check stale chassis
14. Verify routes installed

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
| `route_table_id` | `0` | Kernel route table (0 = main) |
| `reconcile_interval` | `60s` | Periodic reconciliation |
| `dry_run` | `false` | Log only, no changes |
| `cleanup_on_shutdown` | `true` | Remove routes on shutdown |
| `drain_on_shutdown` | `true` | Drain gateways before shutdown |

## Deployment

### System Requirements
- Go 1.25+
- Linux kernel with netlink support
- OVS (Open vSwitch)
- FRRouting (for BGP)
- nftables (for port forwarding)
- Root privileges (CAP_NET_ADMIN)

### Installation
```bash
# Build
make build

# Install
sudo make install

# Configure
sudo cp /etc/ovn-network-agent/config.yaml.sample /etc/ovn-network-agent/config.yaml
sudo vi /etc/ovn-network-agent/config.yaml

# Start
sudo systemctl enable --now ovn-network-agent
```

## Monitoring

### Logging
The agent uses structured logging with slog. Log levels: debug, info, warn, error.

```bash
# View logs
journalctl -u ovn-network-agent -f

# Enable debug logging
ovn-network-agent --log-level debug
```

### Health Checks
- Check systemd status: `systemctl status ovn-network-agent`
- Verify OVN connections: Check logs for connection errors
- Verify routes: `ip route show table <table_id>`
- Verify FRR routes: `vtysh -c "show ip route vrf vrf-provider static"`

## Troubleshooting

### Common Issues

1. **Routes not appearing**: Check `dry_run` mode
2. **OVN connection failed**: Verify DB endpoints
3. **Bridge device missing**: Ensure `br-ex` exists
4. **Permission denied**: Run as root
5. **FRR routes not syncing**: Check vtysh connectivity

### Debug Mode
```bash
# Dry run mode (test without making changes)
ovn-network-agent --dry-run

# Debug logging
ovn-network-agent --log-level debug
```

## Performance

### Scalability
- Tested with thousands of FIPs
- Batch operations (500 per call for FRR)
- Efficient OVSDB monitoring
- Debouncing reduces thrashing

### Resource Usage
- **Memory**: Proportional to route count
- **CPU**: Event-driven, low idle usage
- **Network**: OVSDB monitoring traffic only

### Latency
- **Failover**: <1s for chassisredirect changes
- **Reconciliation**: <5s for full cycle
- **Event processing**: <100ms per event

## Security

### Privilege Requirements
- Must run as root (CAP_NET_ADMIN)
- Can modify kernel routing tables
- Can modify VRF routing tables
- Can install nftables rules

### Network Exposure
- Listens on OVSDB connections
- Configuration contains sensitive data (DB endpoints)

### Recommendations
- Use TLS/SSL for OVSDB connections
- Protect configuration files
- Monitor nftables rules
- Audit VRF access

## Contributing

### Code Structure
```
ovn-network-agent/
├── main.go              # Entry point
├── agent.go             # Main orchestrator
├── config.go            # Configuration
├── ovn.go               # OVN client
├── ovn_gateway.go       # Gateway management
├── routing.go           # Route management
├── routing_linux.go     # Linux-specific routing
├── ovs.go               # OVS flow management
├── nftables.go          # nftables rules
└── tests/               # Test files
```

### Development Workflow
1. Read architecture documentation
2. Understand flow diagrams
3. Review API reference
4. Write tests
5. Submit PR

### Testing
```bash
# Run tests
make test

# Run with coverage
go test -cover ./...

# Run specific test
go test -v -run TestReconcile
```

## Additional Resources

### External Documentation
- [OVN Documentation](https://docs.ovn.org/)
- [Open vSwitch Documentation](https://docs.openvswitch.org/)
- [FRR Documentation](https://docs.frr.org/)
- [nftables Documentation](https://wiki.nftables.org/)

### Related Projects
- [libovsdb](https://github.com/ovn-kubernetes/libovsdb) - OVSDB client library
- [netlink](https://github.com/vishvananda/netlink) - Linux networking library

## Support

### Getting Help
- Check the [Troubleshooting](./ARCHITECTURE.md#troubleshooting) section
- Review [Flow Diagrams](./FLOW_DIAGRAMS.md) for execution flow
- Consult [API Reference](./API_REFERENCE.md) for implementation details
- Enable debug logging for detailed information

### Reporting Issues
When reporting issues, include:
- Agent version
- Configuration (sanitized)
- Log output
- Steps to reproduce
- Expected vs actual behavior

## License

See [LICENSE](../LICENSE) for details.

## Version History

See [CHANGELOG](../CHANGELOG.md) for version history (if available).

---

**Last Updated:** 2026-04-27

**Documentation Version:** 1.0.0
