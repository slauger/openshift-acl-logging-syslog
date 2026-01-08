# Minimal Syslog Server for OpenShift Network Policy Debugging

Temporary syslog server for debugging OpenShift Network Policies with ACL logging.

## Purpose
Minimal external syslog receiver for **temporary** OpenShift Network Policy debugging. Receives OVN-Kubernetes ACL audit logs from OpenShift clusters.

## Quick Setup (5 minutes)

```bash
# 1. Copy config & create directory
cp rsyslog-server.conf /etc/rsyslog.d/10-remote-server.conf
mkdir -p /var/log/remote-hosts

# 2. Restart rsyslog
systemctl restart rsyslog

# 3. Open firewall
firewall-cmd --permanent --add-port=514/udp --add-port=514/tcp
firewall-cmd --reload

# 4. Verify
ss -tulnp | grep 514
```

## Enable Network Policy Logging in OpenShift

### Cluster-wide Logging (all namespaces)

```bash
# Configure ACL audit logging (replace <SYSLOG-IP> with your server)
oc patch network.operator cluster --type=merge -p '{
  "spec": {
    "defaultNetwork": {
      "ovnKubernetesConfig": {
        "policyAuditConfig": {
          "destination": "udp:<SYSLOG-IP>:514",
          "maxFileSize": 50,
          "rateLimit": 20,
          "syslogFacility": "local0"
        }
      }
    }
  }
}'

# Verify configuration
oc get network.operator cluster -o jsonpath='{.spec.defaultNetwork.ovnKubernetesConfig.policyAuditConfig}'
```

### Namespace-specific Logging

To enable logging only for specific namespaces, add the `k8s.ovn.org/acl-logging` annotation:

```bash
# Enable logging for denied traffic in a namespace
oc annotate namespace myapp k8s.ovn.org/acl-logging='{"deny":"alert","allow":"alert"}'

# Only log denied traffic
oc annotate namespace myapp k8s.ovn.org/acl-logging='{"deny":"alert"}'

# Only log allowed traffic
oc annotate namespace myapp k8s.ovn.org/acl-logging='{"allow":"alert"}'

# Disable logging for a namespace
oc annotate namespace myapp k8s.ovn.org/acl-logging-
```

**Log levels:**
- `alert` - Log to syslog (required for external syslog server)
- `notice` - Log only to OVN northbound database
- `info` - Log to both syslog and database
- `debug` - Verbose logging (use with caution)

**Example: Debug specific application**
```bash
# 1. Enable namespace logging
oc annotate namespace frontend k8s.ovn.org/acl-logging='{"deny":"alert","allow":"alert"}'

# 2. Check logs on syslog server
tail -f /var/log/remote-hosts/*/syslog.log | grep "namespace=frontend"

# 3. Disable when done
oc annotate namespace frontend k8s.ovn.org/acl-logging-
```

### Test Network Policy (Generate Logs)

Quick test policy to generate deny logs:

```bash
# Create test namespace
oc create namespace netpol-test

# Enable logging for this namespace
oc annotate namespace netpol-test k8s.ovn.org/acl-logging='{"deny":"debug","allow":"debug"}'

# Apply deny-all policy
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: netpol-test
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Create a test pod
oc run test-pod -n netpol-test --image=registry.access.redhat.com/ubi9/ubi-minimal:latest --command -- sleep 3600

# Try to generate traffic (will be denied)
oc exec -n netpol-test test-pod -- curl -m 5 https://www.redhat.com || echo "Connection denied (expected)"

# Check syslog for deny logs
grep "netpol-test" /var/log/remote-hosts/*/syslog.log

# Cleanup
oc delete namespace netpol-test
```

## View Logs

```bash
# List all nodes sending logs
ls -la /var/log/remote-hosts/

# Tail logs from a node
tail -f /var/log/remote-hosts/<node-hostname>/syslog.log

# Search for denied connections
grep -r "deny" /var/log/remote-hosts/

# Search by namespace
grep -r "namespace=myapp" /var/log/remote-hosts/
```

## Disable When Done

```bash
# Remove ACL logging from OpenShift
oc patch network.operator cluster --type=merge -p '{
  "spec": {
    "defaultNetwork": {
      "ovnKubernetesConfig": {
        "policyAuditConfig": null
      }
    }
  }
}'
```

## Documentation
- [Full installation guide](INSTALLATION.md) - Complete setup with SELinux, log rotation, troubleshooting
- [rsyslog config](rsyslog-server.conf) - Server configuration details

## References
- [OpenShift Network Security - Logging Network Security](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/network_security/logging-network-security)
