# Syslog Server Installation auf RHEL8

## Übersicht
Diese Anleitung beschreibt die Installation und Konfiguration eines zentralen Syslog-Servers mit rsyslog auf RHEL8.
Der Server empfängt Logs über UDP (Port 514) und TCP (Port 514) und speichert die Logs pro Client in separaten Dateien.

## Voraussetzungen
- RHEL8 System mit Root-Rechten
- Firewall-Zugriff für Port 514 (UDP und TCP)

## Installation

### 1. rsyslog installieren/prüfen
rsyslog ist normalerweise bereits auf RHEL8 installiert:

```bash
# Prüfen ob rsyslog installiert ist
rpm -qa | grep rsyslog

# Falls nicht installiert:
dnf install rsyslog -y
```

### 2. rsyslog Service aktivieren
```bash
systemctl enable rsyslog
systemctl start rsyslog
systemctl status rsyslog
```

## Konfiguration

### 3. Backup der Original-Konfiguration erstellen
```bash
cp /etc/rsyslog.conf /etc/rsyslog.conf.backup
```

### 4. rsyslog für Remote-Empfang konfigurieren
Bearbeiten Sie `/etc/rsyslog.conf` oder erstellen Sie eine separate Konfigurationsdatei (empfohlen).

Die Konfigurationsdatei liegt im Repository: `rsyslog-server.conf`

Kopieren Sie diese nach:
```bash
cp rsyslog-server.conf /etc/rsyslog.d/10-remote-server.conf
```

### 5. Log-Verzeichnis erstellen
```bash
mkdir -p /var/log/remote-hosts
chmod 755 /var/log/remote-hosts
```

### 6. rsyslog neu starten
```bash
systemctl restart rsyslog
systemctl status rsyslog
```

### 7. Firewall konfigurieren
```bash
# UDP Port 514 öffnen
firewall-cmd --permanent --add-port=514/udp

# TCP Port 514 öffnen
firewall-cmd --permanent --add-port=514/tcp

# Firewall neu laden
firewall-cmd --reload

# Prüfen
firewall-cmd --list-all
```

### 8. SELinux konfigurieren (falls aktiviert)
```bash
# SELinux Kontext für Log-Verzeichnis setzen
semanage fcontext -a -t var_log_t "/var/log/remote-hosts(/.*)?"
restorecon -R /var/log/remote-hosts

# Erlaube rsyslog auf Port 514 zu lauschen (falls nötig)
semanage port -a -t syslogd_port_t -p udp 514
semanage port -a -t syslogd_port_t -p tcp 514
```

## Testen

### 9. Lokaler Test
```bash
# Test mit logger (UDP)
logger -n 127.0.0.1 -P 514 "Test message from localhost"

# Prüfen ob Log-Datei erstellt wurde
ls -la /var/log/remote-hosts/
cat /var/log/remote-hosts/127.0.0.1/syslog.log
```

### 10. Remote Test von einem Client
Auf dem Client (z.B. einem anderen Linux-System):
```bash
# UDP Test
logger -n <SYSLOG-SERVER-IP> -P 514 "Test message via UDP"

# TCP Test
logger -n <SYSLOG-SERVER-IP> -P 514 -T "Test message via TCP"
```

## Überwachung und Wartung

### Logs prüfen
```bash
# rsyslog Status
systemctl status rsyslog

# rsyslog Logs prüfen
journalctl -u rsyslog -f

# Empfangene Remote-Logs prüfen
ls -lR /var/log/remote-hosts/
```

### Log-Rotation einrichten
Erstellen Sie `/etc/logrotate.d/remote-hosts`:
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

### Ports prüfen
```bash
# Prüfen ob rsyslog auf Port 514 lauscht
ss -tulnp | grep 514
netstat -tulnp | grep 514
```

### Logs kommen nicht an
1. Firewall prüfen: `firewall-cmd --list-all`
2. rsyslog Status: `systemctl status rsyslog`
3. SELinux: `ausearch -m avc -ts recent`
4. rsyslog Debug-Modus: `rsyslogd -N1` (Syntax-Check)
5. Netzwerk-Verbindung: `tcpdump -i any port 514`

### Client-Konfiguration
Die meisten Linux-Systeme können mit rsyslog oder syslog-ng Logs weiterleiten:

In `/etc/rsyslog.conf` auf dem Client:
```
# Alle Logs an zentralen Server senden (UDP)
*.* @<SYSLOG-SERVER-IP>:514

# Oder via TCP (empfohlen)
*.* @@<SYSLOG-SERVER-IP>:514
```

Dann Client rsyslog neu starten:
```bash
systemctl restart rsyslog
```

## Sicherheitshinweise
- Erwägen Sie TLS-Verschlüsselung für TCP (rsyslog unterstützt TLS)
- Beschränken Sie Firewall-Zugriff auf bekannte Client-IPs
- Überwachen Sie Disk-Space für `/var/log/remote-hosts/`
- Implementieren Sie regelmäßige Log-Rotation

## Support
Bei Problemen prüfen Sie:
- Red Hat Enterprise Linux 8 Dokumentation
- rsyslog Dokumentation: https://www.rsyslog.com/doc/
