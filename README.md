# DNS AutoDump - Automated Network Diagnostics

This tool automatically captures network diagnostics (tcpdump, strace, etc.) when DNS resolution failures are detected in an OpenShift cluster.

## Overview

The DNS AutoDump solution consists of two DaemonSets:
- **Tester**: Continuously checks DNS resolution for a configurable list of URLs
- **Catcher**: Monitors the tester and automatically captures network diagnostics when any URL check fails

Based on Red Hat Article: https://access.redhat.com/articles/7054826

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│ ConfigMap: tcpdump-on-dns-urls                          │
│ - List of URLs to check                                 │
└─────────────────────────────────────────────────────────┘
                    │
                    ├─────────────┬─────────────────┐
                    ▼             ▼                 ▼
        ┌─────────────────┐  ┌──────────────┐  ┌──────────────┐
        │ Tester Pod      │  │ Tester Pod   │  │ Tester Pod   │
        │ (Node 1)        │  │ (Node 2)     │  │ (Node 3)     │
        │                 │  │              │  │              │
        │ Checks URLs     │  │ Checks URLs  │  │ Checks URLs  │
        │ every 5 sec     │  │ every 5 sec  │  │ every 5 sec  │
        └─────────────────┘  └──────────────┘  └──────────────┘
                    │             │                 │
                    │ Failure?    │ Failure?        │ Failure?
                    ▼             ▼                 ▼
        ┌─────────────────┐  ┌──────────────┐  ┌──────────────┐
        │ Catcher Pod     │  │ Catcher Pod  │  │ Catcher Pod  │
        │ (Node 1)        │  │ (Node 2)     │  │ (Node 3)     │
        │                 │  │              │  │              │
        │ Triggers        │  │ Triggers     │  │ Triggers     │
        │ tcpdump + logs  │  │ tcpdump +log │  │ tcpdump +log │
        └─────────────────┘  └──────────────┘  └──────────────┘
                    │             │                 │
                    └─────────────┴─────────────────┘
                                  ▼
                    ┌──────────────────────────────┐
                    │ /var/tmp on each node        │
                    │ - node_*.pcap                │
                    │ - pod_*.pcap                 │
                    │ - pod_strace-*.log           │
                    │ - pod_ss_-a-*.log            │
                    │ - pod_nstat_-Zas-*.log       │
                    │ - pod_snmp-*.log             │
                    │ - node_top-*.log             │
                    │ - failed_urls-*.log          │
                    └──────────────────────────────┘
```

## Prerequisites

- OpenShift 4.x cluster
- Cluster admin privileges
- `jq` is available in the container image (or will be installed from host)

## Deployment

### 1. Deploy the resources

```bash
oc apply -f dns_autodump.yaml
```

This creates:
- Namespace: `tcpdump-on-dns`
- Security Context Constraints (privileged and ptrace)
- Service Accounts
- ConfigMap with URL list
- Two DaemonSets (tester and catcher)

### 2. Enable monitoring on specific nodes

The DaemonSets only run on labeled nodes:

```bash
# Label a single node
oc label node <node-name> tcpdump-on-dns-enable=true

# Label multiple nodes
oc label node worker-0 worker-1 worker-2 tcpdump-on-dns-enable=true

# Label all worker nodes
oc label node -l node-role.kubernetes.io/worker tcpdump-on-dns-enable=true
```

### 3. Verify deployment

```bash
# Check pods are running
oc get pods -n tcpdump-on-dns

# Expected output:
# NAME                  READY   STATUS    RESTARTS   AGE
# logs-catcher-xxxxx    1/1     Running   0          1m
# tester-xxxxx          1/1     Running   0          1m
```

## Configuration

### Adding/Modifying URLs to Check

Edit the ConfigMap to add or remove URLs:

```bash
oc edit configmap tcpdump-on-dns-urls -n tcpdump-on-dns
```

Example configuration:

```yaml
data:
  urls.txt: |
    kubernetes.default.svc.cluster.local
    openshift.default.svc.cluster.local
    api.openshift.example.com
    oauth-openshift.apps.example.com
    console-openshift-console.apps.example.com
    registry.redhat.io
    google.com
    # Lines starting with # are comments
    # One URL per line
```

**Note:** After updating the ConfigMap, the pods will pick up changes automatically (may take up to 60 seconds due to kubelet sync).

### Adjusting Check Interval

To change the check frequency, edit the tester DaemonSet:

```bash
oc edit daemonset tester -n tcpdump-on-dns
```

Modify the `sleep` value (default is 5 seconds):

```yaml
sleep 5 ## Execute the script every 5 seconds
```

## Monitoring

### Watch tester logs

```bash
# Follow logs from all tester pods
oc logs -n tcpdump-on-dns -l app=tcpdump-on-dns-tester -f

# Example output:
# 2026-03-25-14:23:10 OK: kubernetes.default.svc.cluster.local
# 2026-03-25-14:23:10 OK: openshift.default.svc.cluster.local
# 2026-03-25-14:23:10 Summary: All URLs resolved successfully
# 2026-03-25-14:23:15 FAILED: kubernetes.default.svc.cluster.local
# 2026-03-25-14:23:15 OK: openshift.default.svc.cluster.local
# 2026-03-25-14:23:15 Summary: Failed URLs: kubernetes.default.svc.cluster.local
```

### Watch catcher logs

```bash
# Follow logs from all catcher pods
oc logs -n tcpdump-on-dns -l app=tcpdump-on-dns -f

# Example output when failure detected:
# 2026-03-25-14:23:15 Log collection: URL check(s) failed. Failed URLs: kubernetes.default.svc.cluster.local. Collecting the logs...
# 2026-03-25-14:25:16 Log collection: completed.
```

## Collected Data

When a URL check fails, the following diagnostics are automatically captured:

| File | Description | Location |
|------|-------------|----------|
| `node_<nodename>-<timestamp>.pcap` | Network capture from host (DNS, ICMP, ARP) | `/var/tmp` |
| `pod_<nodename>-<timestamp>.pcap` | Network capture from pod namespace (all traffic) | `/var/tmp` |
| `pod_strace-<nodename>-<timestamp>.log` | System call trace of curl to kubernetes API | `/var/tmp` |
| `pod_ss_-a-<nodename>-<timestamp>.log` | Socket statistics from tester pod | `/var/tmp` |
| `pod_nstat_-Zas-<nodename>-<timestamp>.log` | Network statistics from tester pod | `/var/tmp` |
| `pod_snmp-<nodename>-<timestamp>.log` | SNMP statistics from tester pod | `/var/tmp` |
| `node_top-<nodename>-<timestamp>.log` | Process snapshot from host | `/var/tmp` |
| `failed_urls-<nodename>-<timestamp>.log` | List of URLs that failed at capture time | `/var/tmp` |

### Accessing Collected Data

**Option 1: Debug pod**

```bash
oc debug node/<node-name>
chroot /host
cd /var/tmp
ls -lh *$(date +%F)*
```

**Option 2: Copy from node**

```bash
# List files
oc debug node/<node-name> -- chroot /host ls -lh /var/tmp/

# Copy specific file
oc debug node/<node-name> -- chroot /host cat /var/tmp/node_worker-0-2026-03-25-14:23:15.pcap > node_capture.pcap
```

**Option 3: Direct node access (if available)**

```bash
ssh core@<node-ip>
sudo ls -lh /var/tmp/*$(date +%F)*
```

## Storage Management

### Automatic Storage Protection

The catcher script includes automatic storage protection:
- Checks disk usage before capturing
- Refuses to capture if `/var/tmp` is more than 75% full
- Logs error message: `"Impossible to continue: Usage of /var/tmp above 75%."`

### Manual Cleanup

```bash
# List all captured files
oc debug node/<node-name> -- chroot /host ls -lh /var/tmp/

# Remove old captures (older than 7 days)
oc debug node/<node-name> -- chroot /host find /var/tmp -name "node_*" -mtime +7 -delete
oc debug node/<node-name> -- chroot /host find /var/tmp -name "pod_*" -mtime +7 -delete

# Remove all captures
oc debug node/<node-name> -- chroot /host rm -f /var/tmp/node_* /var/tmp/pod_* /var/tmp/failed_urls-*
```

## Troubleshooting

### Pods not starting

**Check node labels:**
```bash
oc get nodes --show-labels | grep tcpdump-on-dns
```

**Check pod status:**
```bash
oc describe pod -n tcpdump-on-dns <pod-name>
```

### Tester container not ready error

If catcher logs show: `"Impossible to continue: Tester container not ready."`

**Check tester pod:**
```bash
oc get pods -n tcpdump-on-dns -l app=tcpdump-on-dns-tester
oc logs -n tcpdump-on-dns -l app=tcpdump-on-dns-tester
```

**Common causes:**
- Tester pod not running yet (wait 30 seconds)
- Tester pod crashed (check logs)
- Node labeled but pods scheduled elsewhere

### No captures being generated

**Verify URL checks are failing:**
```bash
oc logs -n tcpdump-on-dns -l app=tcpdump-on-dns-tester | grep FAILED
```

**Check catcher is running:**
```bash
oc logs -n tcpdump-on-dns -l app=tcpdump-on-dns -f
```

**Manually test URL resolution in tester pod:**
```bash
oc rsh -n tcpdump-on-dns <tester-pod-name>
getent hosts kubernetes.default.svc.cluster.local
```

### Disk space issues

**Check disk usage:**
```bash
oc debug node/<node-name> -- chroot /host df -h /var/tmp
```

**If above 75%, clean old captures:**
```bash
oc debug node/<node-name> -- chroot /host find /var/tmp -name "*.pcap" -mtime +1 -delete
```

## Cleanup

### Remove from specific nodes

```bash
# Remove label from nodes
oc label node <node-name> tcpdump-on-dns-enable-

# Pods will be automatically terminated
```

### Complete removal

```bash
# Delete all resources
oc delete project tcpdump-on-dns

# This removes:
# - All pods
# - All DaemonSets
# - ConfigMap
# - Service Accounts
# - Role Bindings
```

**Note:** This does NOT remove:
- SecurityContextConstraints (cluster-scoped)
- ClusterRole (cluster-scoped)
- Captured data on nodes (`/var/tmp/`)

**Manual cleanup of cluster-scoped resources:**
```bash
oc delete scc tcpdump-on-dns-privileged-scc tcpdump-on-dns-ptrace-scc
oc delete clusterrole system:openshift:scc:tcpdump-on-dns-privileged-scc system:openshift:scc:tcpdump-on-dns-ptrace-scc
```

**Manual cleanup of captured data:**
```bash
for node in $(oc get nodes -o name); do
  oc debug $node -- chroot /host rm -f /var/tmp/node_* /var/tmp/pod_* /var/tmp/failed_urls-*
done
```

## Best Practices

1. **Start with a small set of nodes**
   - Test on 1-2 nodes first
   - Verify captures are working
   - Then expand to more nodes

2. **Adjust check interval based on needs**
   - Default 5 seconds is good for most cases
   - Increase to 10-30 seconds for less frequent checks
   - Decrease to 1-2 seconds for high-frequency monitoring

3. **Monitor disk usage regularly**
   - Each capture session creates ~10-50MB of data
   - Set up alerts for `/var/tmp` usage
   - Implement automated cleanup jobs

4. **Limit URL list size**
   - 5-10 URLs is usually sufficient
   - Too many URLs increase check time
   - Focus on critical services

5. **Use meaningful URLs**
   - Internal cluster services (kubernetes.default, openshift.default)
   - Application-specific endpoints
   - External dependencies your apps rely on

## Security Considerations

- The catcher pod runs with **privileged** SCC (required for tcpdump and host access)
- The tester pod runs with **ptrace** SCC (required for strace)
- Both pods have access to host network namespace
- Captured data may contain sensitive network traffic
- Limit access to `/var/tmp` on nodes
- Clean up captures promptly after analysis

## Advanced Usage

### Custom capture duration

Edit the script in the Secret to change tcpdump duration (default 2 minutes):

```bash
oc edit secret tcpdump-on-dns-script -n tcpdump-on-dns
```

Modify the timeout values in the script:
```bash
timeout 2m tcpdump ...
# Change to:
timeout 5m tcpdump ...
```

### Custom tcpdump filters

Edit the script to add specific tcpdump filters:

```bash
# Current filter: port 53 or port 5353 or icmp or arp
timeout 2m tcpdump -B 20480 -s 260 -nni any -w ${PCAPFILENAME} port 53 or port 5353 or icmp or arp

# Add HTTP/HTTPS traffic:
timeout 2m tcpdump -B 20480 -s 260 -nni any -w ${PCAPFILENAME} port 53 or port 5353 or port 80 or port 443 or icmp or arp
```

### Integration with must-gather

Captures are stored in `/var/tmp` which is collected by must-gather:

```bash
oc adm must-gather
```

The captured files will be included in the must-gather archive under the node directories.

## Support

For issues or questions:
1. Check the troubleshooting section above
2. Review Red Hat Article: https://access.redhat.com/articles/7054826
3. Open a support case with Red Hat

## License

This tool is provided as-is for diagnostic purposes in OpenShift environments.
