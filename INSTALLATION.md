# Syslog Server Installation on RHEL8

## Overview
This guide describes the installation and configuration of a central syslog server with rsyslog on RHEL8.
The server receives logs via UDP (port 514) and TCP (port 514) and stores the logs per client in separate files.

## Prerequisites
- RHEL8 system with root privileges
- Firewall access for port 514 (UDP and TCP)

## Installation

### 1. Install/check rsyslog
rsyslog is normally already installed on RHEL8:

```bash
# Check if rsyslog is installed
rpm -qa | grep rsyslog

# If not installed:
dnf install rsyslog -y
```

### 2. Enable rsyslog service
```bash
systemctl enable rsyslog
systemctl start rsyslog
systemctl status rsyslog
```

## Configuration

### 3. Create backup of original configuration
```bash
cp /etc/rsyslog.conf /etc/rsyslog.conf.backup
```

### 4. Configure rsyslog for remote reception
Edit `/etc/rsyslog.conf` or create a separate configuration file (recommended).

The configuration file is in the repository: `rsyslog-server.conf`

Copy it to:
```bash
cp rsyslog-server.conf /etc/rsyslog.d/10-remote-server.conf
```

### 5. Create log directory
```bash
mkdir -p /var/log/remote-hosts
chmod 755 /var/log/remote-hosts
```

### 6. Restart rsyslog
```bash
systemctl restart rsyslog
systemctl status rsyslog
```

### 7. Configure firewall
```bash
# Open UDP port 514
firewall-cmd --permanent --add-port=514/udp

# Open TCP port 514
firewall-cmd --permanent --add-port=514/tcp

# Reload firewall
firewall-cmd --reload

# Verify
firewall-cmd --list-all
```

### 8. Configure SELinux (if enabled)
```bash
# Set SELinux context for log directory
semanage fcontext -a -t var_log_t "/var/log/remote-hosts(/.*)?"
restorecon -R /var/log/remote-hosts

# Allow rsyslog to listen on port 514 (if necessary)
semanage port -a -t syslogd_port_t -p udp 514
semanage port -a -t syslogd_port_t -p tcp 514
```

## Testing

### 9. Local test
```bash
# Test with logger (UDP)
logger -n 127.0.0.1 -P 514 "Test message from localhost"

# Check if log file was created
ls -la /var/log/remote-hosts/
cat /var/log/remote-hosts/127.0.0.1/syslog.log
```

### 10. Remote test from a client
On the client (e.g., another Linux system):
```bash
# UDP test
logger -n <SYSLOG-SERVER-IP> -P 514 "Test message via UDP"

# TCP test
logger -n <SYSLOG-SERVER-IP> -P 514 -T "Test message via TCP"
```

## Monitoring and Maintenance

### Check logs
```bash
# rsyslog status
systemctl status rsyslog

# Check rsyslog logs
journalctl -u rsyslog -f

# Check received remote logs
ls -lR /var/log/remote-hosts/
```

### Set up log rotation
Create `/etc/logrotate.d/remote-hosts`:
```
/var/log/remote-hosts/*/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0644 root root
    sharedscripts
    postrotate
        /bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true
    endscript
}
```

## Troubleshooting

### Check ports
```bash
# Check if rsyslog is listening on port 514
ss -tulnp | grep 514
netstat -tulnp | grep 514
```

### Logs are not arriving
1. Check firewall: `firewall-cmd --list-all`
2. rsyslog status: `systemctl status rsyslog`
3. SELinux: `ausearch -m avc -ts recent`
4. rsyslog debug mode: `rsyslogd -N1` (syntax check)
5. Network connection: `tcpdump -i any port 514`

### Client configuration
Most Linux systems can forward logs with rsyslog or syslog-ng:

In `/etc/rsyslog.conf` on the client:
```
# Send all logs to central server (UDP)
*.* @<SYSLOG-SERVER-IP>:514

# Or via TCP (recommended)
*.* @@<SYSLOG-SERVER-IP>:514
```

Then restart client rsyslog:
```bash
systemctl restart rsyslog
```

## Security Notes
- Consider TLS encryption for TCP (rsyslog supports TLS)
- Restrict firewall access to known client IPs
- Monitor disk space for `/var/log/remote-hosts/`
- Implement regular log rotation

## Support
For issues, check:
- Red Hat Enterprise Linux 8 Documentation
- rsyslog Documentation: https://www.rsyslog.com/doc/
