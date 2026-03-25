# Auto-tcpdump

Automated tcpdump capture tools for OpenShift/Kubernetes clusters. These tools automatically trigger network packet captures and diagnostic collection when specific failure conditions are detected.

## Available Tools

### [DNS AutoDump](dns/)
Automatically captures tcpdump and system diagnostics when DNS resolution failures are detected.

**Use case:** Troubleshooting intermittent DNS issues in OpenShift clusters

**Features:**
- Configurable URL monitoring list
- Automatic tcpdump capture on DNS failures
- Comprehensive diagnostics (pcap, strace, nstat, ss, snmp, top)
- Storage protection (75% threshold)
- DaemonSet deployment for node-level monitoring

**Quick start:**
```bash
oc apply -f dns/dns_autodump.yaml
oc label node <node-name> tcpdump-on-dns-enable=true
```

[Full documentation →](dns/README.md)

---

## General Concept

Each tool in this repository follows the same pattern:

1. **Tester Pod** - Continuously monitors for specific failure conditions
2. **Catcher Pod** - Triggers automatic diagnostic collection when failures are detected
3. **ConfigMap** - Customizable test parameters
4. **Automatic Cleanup** - Storage protection and resource management

## Contributing

Feel free to add new auto-tcpdump scenarios:
- HTTP endpoint failures
- Database connection issues
- Service mesh timeouts
- Certificate expiration events
- Network policy blocks

## Requirements

- OpenShift 4.x or Kubernetes 1.20+
- Cluster admin privileges (for SCC/PSP)
- Sufficient node storage for captures

## Support

These tools are provided as-is for diagnostic purposes. For production use, review and adjust:
- Storage thresholds
- Capture duration
- Check intervals
- Security constraints

## License

This collection is provided for diagnostic and troubleshooting purposes in OpenShift/Kubernetes environments.
