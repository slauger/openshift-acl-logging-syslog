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
