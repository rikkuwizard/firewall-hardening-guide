# Firewall Hardening Guide - Detailed Procedures

## 📋 Table of Contents

1. [Change Default Credentials](#1-change-default-credentials)
2. [Restrict Admin Access](#2-restrict-admin-access)
3. [Disable Insecure Services](#3-disable-insecure-services)
4. [Enable Logging](#4-enable-logging)
5. [Update Firmware](#5-update-firmware)
6. [Remove Unused Rules](#6-remove-unused-rules)
7. [Backup Configurations](#7-backup-configurations)
8. [Hardening Checklist](#hardening-checklist)

---

## 1. Change Default Credentials

### Why It Matters
Default credentials are publicly known and are one of the first attack vectors used by threat actors. Changing them is the first line of defense.

### Implementation Steps

1. **Log into firewall management interface** (web UI or SSH)
2. **Navigate to User/Admin Settings** (varies by platform)
3. **Create a strong password** following these requirements:
   - Minimum 14 characters
   - Mix of uppercase, lowercase, numbers, and special characters
   - Avoid common words or patterns
   - Example: `Fw#Secure2026$Net`
4. **Change the admin password**
5. **Create additional admin accounts** with unique credentials for each administrator
6. **Document and securely store** credentials in a password manager (e.g., 1Password, LastPass, Vault)

### Verification

```bash
# SSH to firewall and verify new credentials work
ssh admin@firewall.example.com

# Attempt login with old credentials (should fail)
# If successful, repeat password change procedure
```

---

## 2. Restrict Admin Access

### Why It Matters
Unrestricted management access increases the risk of unauthorized configuration changes and credential compromise.

### Implementation Steps

1. **Identify trusted IP addresses/VLANs**
   - Administration VLAN
   - Management server IPs
   - NOC workstations
   
2. **Access Administration Settings**
3. **Configure Management Access Restrictions**
   ```
   - Limit SSH access to: 10.0.1.0/24
   - Limit HTTPS access to: 192.168.100.0/24
   - Disable HTTP management entirely
   ```

4. **Implement Role-Based Access Control (RBAC)**
   - Read-Only roles for monitoring teams
   - Admin roles only for authorized personnel
   - Auditor roles for compliance teams

5. **Enable MFA (Multi-Factor Authentication)** where available
   - RADIUS integration
   - TACACS+ authentication
   - Hardware tokens or software 2FA

### Verification

```bash
# Test access from allowed IP (should succeed)
ssh admin@firewall.example.com

# Test access from restricted IP (should fail)
ssh admin@firewall.example.com  # From untrusted network
```

---

## 3. Disable Insecure Services

### Why It Matters
Insecure protocols transmit credentials and data in plaintext, making them vulnerable to interception and man-in-the-middle attacks.

### Insecure vs. Secure Protocols

| Service | Port | Security | Status | Alternative |
|---------|------|----------|--------|-------------|
| Telnet | 23 | None (Plaintext) | ❌ Disable | SSH (22) |
| HTTP | 80 | None (Plaintext) | ❌ Disable | HTTPS (443) |
| FTP | 21 | None (Plaintext) | ❌ Disable | SFTP/SCP |
| SSH | 22 | Encrypted | ✅ Enable | - |
| HTTPS | 443 | Encrypted (TLS) | ✅ Enable | - |

### Implementation Steps

1. **Access Device Services Settings**
2. **Disable the following services:**
   ```
   - Telnet: DISABLED
   - HTTP (Web Management): DISABLED
   - FTP: DISABLED
   - SNMP v1/v2: DISABLED (enable v3 only if needed)
   ```

3. **Enable secure alternatives:**
   ```
   - SSH: ENABLED (port 22)
   - HTTPS: ENABLED (port 443 with valid certificate)
   - SFTP: ENABLED if file transfer needed
   ```

4. **Configure certificate settings**
   - Use valid, signed certificates (not self-signed where possible)
   - Set certificate renewal reminders
   - Document certificate expiration dates

### Verification

```bash
# Verify Telnet is disabled (connection should timeout/refuse)
telnet firewall.example.com 23

# Verify HTTP is disabled (connection should refuse)
curl http://firewall.example.com:80

# Verify SSH is available (should connect)
ssh admin@firewall.example.com
```

---

## 4. Enable Logging

### Why It Matters
Logs provide visibility into security events, enable threat detection, and are essential for compliance and incident investigation.

### What to Log

| Event Type | Importance | Details |
|-----------|-----------|---------|
| **Denied Traffic** | Critical | All dropped/rejected connections |
| **Config Changes** | Critical | Who changed what, when, and why |
| **Authentication** | Critical | Login attempts (success & failure) |
| **System Events** | High | Restarts, updates, errors |
| **VPN Activity** | High | VPN connections, disconnects, failures |

### Implementation Steps

1. **Configure Traffic Logging**
   ```
   Enable: Log all denied traffic
   Include: Source IP, Destination IP, Port, Protocol, Action
   Retention: Minimum 90 days
   ```

2. **Configure Admin/Config Logging**
   ```
   Enable: Log all configuration changes
   Include: Administrator, timestamp, change details
   Syslog Server: Configure external syslog server
   ```

3. **Configure Authentication Logging**
   ```
   Enable: All login attempts
   Include: Username, source IP, success/failure, timestamp
   Alert on: Failed login attempts (>3 in 5 minutes)
   ```

4. **Configure Syslog/Remote Logging**
   ```
   Primary Syslog Server: syslog.example.com
   Secondary Syslog Server: syslog-backup.example.com
   Protocol: UDP 514 or TCP 601
   ```

### Verification

```bash
# Check local log storage
ssh admin@firewall.example.com
show log

# Verify syslog is receiving events
tail -f /var/log/syslog | grep firewall

# Test by making a denied connection
curl http://firewall.example.com:22  # Should be logged as denied
```

---

## 5. Update Firmware

### Why It Matters
Firmware updates patch critical security vulnerabilities, add security features, and improve stability.

### Implementation Steps

1. **Check Current Firmware Version**
   ```bash
   ssh admin@firewall.example.com
   show system version
   ```

2. **Download Latest Firmware**
   - Visit manufacturer support portal
   - Verify checksum/digital signature
   - Review release notes for compatibility

3. **Schedule Maintenance Window**
   - Notify stakeholders of planned downtime
   - Backup current configuration (see section 7)
   - Consider failover/HA setup if available

4. **Install Firmware**
   ```
   Method 1: Web UI
   - Administration > Firmware > Upload & Install
   
   Method 2: CLI
   - firmware upload {filename}
   - firmware install {filename}
   ```

5. **Reboot and Verify**
   ```bash
   system reboot
   # Wait 5-10 minutes
   ssh admin@firewall.example.com
   show system version  # Verify new version
   ```

### Verification

```bash
# Confirm firmware version
show system version

# Verify firewall functionality
show system status
show rule base
show vpn connections
```

---

## 6. Remove Unused Rules

### Why It Matters
Overly permissive or unused rules increase attack surface and make it harder to maintain security policies.

### Implementation Steps

1. **Audit Rule Base**
   - Export current rules: `export rule-base to firewall-rules.csv`
   - Review each rule for business justification
   - Identify unused or overly permissive rules

2. **Remove ANY-ANY Rules**
   ```
   These rules allow all traffic:
   - Source: ANY
   - Destination: ANY
   - Service: ANY
   - Action: ALLOW
   
   Replace with specific rules or justify in writing
   ```

3. **Consolidate Redundant Rules**
   - Combine similar rules where possible
   - Remove duplicate entries
   - Organize by logical grouping

4. **Implement Rule Maintenance Process**
   ```
   Quarterly Review:
   - Q1: Review and clean rule base
   - Q2: Remove rules with no traffic
   - Q3: Audit for compliance
   - Q4: Update documentation
   ```

### Verification

```bash
# Check rule hit counts
show rule-base statistics

# Identify unused rules (zero hits in 90 days)
show rule-base unused --days 90

# Review final rule base
show rule-base summary
```

---

## 7. Backup Configurations

### Why It Matters
Regular backups ensure you can quickly recover from accidental misconfiguration, hardware failure, or ransomware attacks.

### Implementation Steps

1. **Create Backup Schedule**
   ```
   Frequency: Daily at 2 AM
   Retention: Keep 30 daily backups + 12 monthly backups
   Location: Offsite backup server
   Encryption: AES-256 with strong passphrase
   ```

2. **Configure Automated Backups**
   ```bash
   # CLI command to create backup
   backup configuration filename:fw-backup-$(date +%Y%m%d).bin
   
   # SCP backup to remote server
   scp admin@firewall.example.com:/backup/fw-config.bin /backups/offsite/
   ```

3. **Test Restore Process**
   - Monthly: Perform test restore on isolated lab firewall
   - Verify all configurations are restored
   - Document any issues or steps needed

4. **Secure Backup Storage**
   - Store backups on separate server/network
   - Encrypt backups at rest and in transit
   - Restrict access to authorized personnel only
   - Use versioning to track changes over time

### Verification

```bash
# Create a test backup
backup configuration filename:test-backup.bin

# Verify backup integrity
show backup:test-backup.bin
size: 2.4 MB
created: 2026-04-28 02:00:00
checksum: a3f9b2c1e4d7...

# Perform test restore (in lab environment)
restore configuration filename:test-backup.bin
```

---

## ✅ Hardening Checklist

Use this checklist to track your firewall hardening progress:

### Credentials & Access
- [ ] Default credentials changed
- [ ] Strong password policy implemented
- [ ] Admin credentials documented securely
- [ ] Additional admin accounts created with unique credentials
- [ ] MFA enabled for admin access
- [ ] Management access restricted to trusted IPs/VLANs

### Services & Protocols
- [ ] Telnet disabled
- [ ] HTTP (web management) disabled
- [ ] FTP disabled
- [ ] SSH enabled and tested
- [ ] HTTPS enabled with valid certificate
- [ ] SNMP v3 only (v1/v2 disabled if SNMP not needed)

### Logging & Monitoring
- [ ] Traffic logging enabled
- [ ] Config change logging enabled
- [ ] Authentication logging enabled
- [ ] Syslog/remote logging configured
- [ ] Log retention policy set (minimum 90 days)
- [ ] Centralized log collection tested

### Rules & Policies
- [ ] Rule base audited
- [ ] ANY-ANY rules removed or justified
- [ ] Unused rules removed
- [ ] Rules consolidated and organized
- [ ] Rule documentation updated

### Firmware & Updates
- [ ] Latest firmware version installed
- [ ] Firmware update procedure documented
- [ ] Security update schedule established

### Backup & Recovery
- [ ] Automated backup schedule configured
- [ ] Backups encrypted and stored offsite
- [ ] Restore procedure tested
- [ ] Disaster recovery plan created

---

**Last Updated:** 2026-04-28  
**Status:** Complete ✅

For additional resources and platform-specific guides, see the main [README.md](../README.md).
