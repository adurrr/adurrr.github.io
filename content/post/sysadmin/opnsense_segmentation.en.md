+++
author = "Adur"
title = "OPNsense: network segmentation, VLANs and hardening"
date = "2026-04-03"
description = "Native encrypted backups, Zenarmor, VLANs for network segmentation into zones, and hardening practices to take OPNsense to a serious security level."
tags = [
    "opnsense",
    "firewall",
    "vlans",
    "zenarmor",
    "networking",
    "hardening",
    "homelab"
]
categories = [
    "sysadmin"
]
series = ["OPNsense"]
toc = true
+++

## Native encrypted backups

Losing the OPNsense configuration after hours of tweaking is the kind of disaster that only happens once. After that incident, automatic backups are configured.

OPNsense has a native backup system that exports all configuration to an XML file. Since version 24.1, these backups can be encrypted directly from the web interface.

### Encrypted backup configuration

In **System > Configuration > Backups**:

1. Go to the **Google Drive / Nextcloud** section if you want remote backup, or stick with local backup.
2. Check the **Encrypt backup** box and enter an encryption password. This password is independent of system credentials. Store it in a password manager, because without it the backup is unrecoverable.
3. In the automatic backup section (**Scheduled**), configure the frequency. A daily backup is reasonable for most environments.

### Manual backup from the interface

In **System > Configuration > Backups**, the **Download configuration** button generates an XML with the current configuration. If encryption was selected, the downloaded file will be encrypted with AES-256-CBC.

### Backup from the console

To automate backups from the command line:

```bash
# Export the configuration
cp /conf/config.xml /root/backup_$(date +%Y%m%d).xml

# Encrypt with OpenSSL
openssl enc -aes-256-cbc -salt -pbkdf2 \
  -in /root/backup_$(date +%Y%m%d).xml \
  -out /root/backup_$(date +%Y%m%d).xml.enc

# Remove the unencrypted file
rm /root/backup_$(date +%Y%m%d).xml
```

It is recommended to copy encrypted backups to external storage: a NAS, an S3 bucket, or even a private Git repository (the encrypted XML is small, a few hundred KB).

### Restoration

To restore, go to **System > Configuration > Backups**, upload the file and, if encrypted, enter the password. OPNsense applies the configuration and restarts the affected services.

## Static IP assignment

There are devices that need to always have the same IP: servers, NAS, printers, surveillance cameras. This can be done in two ways: configuring a static IP on the device itself or, which is cleaner, assigning DHCP reservations in OPNsense.

### DHCP reservations (the recommended way)

In **Services > DHCPv4 > [interface]**:

1. Go to the **DHCP Static Mappings** section.
2. Add a new entry with:
   - **MAC Address**: the MAC address of the device.
   - **IP Address**: the IP you want to always assign.
   - **Hostname**: a descriptive name.

The advantage of doing it this way is that management is centralized in OPNsense. If you change routers tomorrow, devices don't need to be reconfigured.

### Range convention

A convention that works well for organizing the network:

| Range | Usage |
|-------|-----|
| .1 | Gateway (OPNsense) |
| .2 - .19 | Infrastructure (switches, APs, NAS) |
| .20 - .49 | Servers and services |
| .50 - .99 | Fixed IP devices (printers, cameras) |
| .100 - .254 | Dynamic DHCP pool |

This means that just by seeing a device's IP, you already know which category it falls into.

## Zenarmor (Sensei)

Zenarmor is a deep packet inspection (DPI) plugin for OPNsense. It goes beyond what Suricata or CrowdSec do because it inspects traffic at the application level: it can distinguish between Netflix and YouTube, between Telegram and WhatsApp, between legitimate traffic and potentially dangerous applications.

### What it does exactly

- **Application-based traffic classification**: identifies more than 300 applications and protocols.
- **Content filtering by categories**: allows blocking entire categories (gambling, malware, adult content) without needing to maintain lists manually.
- **Encrypted traffic analysis (TLS)**: Zenarmor can classify HTTPS traffic without decrypting it, using metadata like SNI, JA3 fingerprints, and connection patterns.
- **Detailed reporting**: dashboards with consumption by device, application, and category.

### Installation

In **System > Firmware > Plugins**, search for `os-sunnyvalley` and install. After installation, Zenarmor appears in the main menu.

When starting Zenarmor for the first time, a configuration wizard runs:

1. **Deployment mode**: choose **Routed Mode** to inspect all traffic passing through OPNsense. **Bridge** mode is for specific cases.
2. **Database engine**: Zenarmor uses a local database for logs. For modest hardware (N100), select **SQLite**. For more powerful hardware, **Elasticsearch** gives better query performance.
3. **Interfaces to protect**: select the LAN interfaces you want to inspect.
4. **Default policy**: start with a permissive policy (monitor only) and adjust after seeing actual traffic.

### Policy configuration

In **Zenarmor > Policies**:

Policies are applied by interface or by device group. A reasonable configuration:

**General policy (main LAN)**:
- Block categories: Malware, Phishing, Cryptomining, C2 (Command & Control).
- Monitor but allow: Streaming, Social Media, Gaming.
- Allow everything else.

**Policy for IoT** (we'll create this with VLANs later):
- Block everything except the necessary domains for each device.
- IoT devices should not be able to access the internet freely.

**Policy for guests**:
- Block: P2P, Tor, VPN (to avoid filter bypass).
- Limit bandwidth per device.

### Difference with Suricata and CrowdSec

| Feature | Suricata (IDS/IPS) | CrowdSec | Zenarmor |
|---|---|---|---|
| **Inspection** | Packets and signatures | Logs and patterns | Application (DPI) |
| **What it detects** | Exploits, malware, C2 | Brute force, scans | Applications, categories |
| **Blocking** | By signature/rule | By IP (temp ban) | By application/category |
| **Main resource** | CPU (high) | CPU (low) | CPU (medium) + RAM |
| **Complementary** | Yes | Yes | Yes |

All three complement each other. Suricata looks for known threats in traffic. CrowdSec detects malicious behavior in logs and shares intelligence. Zenarmor classifies and filters at the application level. Using them together gives security coverage that is hard to beat on home equipment.

## Disable insecure SSH

SSH is the usual way to access the OPNsense console remotely. But the default configuration has aspects that should be changed.

### Secure SSH configuration

In **System > Settings > Administration**, SSH section:

1. **Disable root login via SSH**. Create a specific user with SSH access and sudo permissions.
2. **Change the default port**. Port 22 is the first one bots scan. Change to a high port (for example, 2222 or something less predictable).
3. **Disable password authentication**. Use exclusively SSH keys:

```bash
# On your local machine, generate a key pair if you don't have one
ssh-keygen -t ed25519 -C "opnsense-admin"

# Copy the public key
cat ~/.ssh/id_ed25519.pub
```

4. Paste the public key in the user's profile at **System > Access > Users > [user] > Authorized Keys**.
5. In the SSH configuration, uncheck **Permit Password Login**.

### SSH access restriction by firewall

Create a firewall rule that only allows SSH from specific IPs:

In **Firewall > Rules > LAN**, create a rule:
- Action: Pass
- Protocol: TCP
- Source: alias with admin IPs
- Destination: This Firewall
- Destination port: the configured SSH port

And another rule that blocks SSH from any other source.

## VLANs: network segmentation

Segmentation with VLANs is probably the most important change you can make to home network security. Without segmentation, a compromised WiFi bulb has direct access to the NAS with family photos. With VLANs, each type of device lives in its own isolated segment.

### VLAN design

| VLAN ID | Name | Subnet | Purpose |
|---------|--------|--------|-----------|
| 10 | Main | 192.168.10.0/24 | Trusted devices: laptops, desktops, personal mobiles |
| 20 | Guests | 192.168.20.0/24 | Guest devices, no access to internal network |
| 30 | IoT | 192.168.30.0/24 | IoT devices: cameras, sensors, bulbs, vacuums |
| 40 | Servers | 192.168.40.0/24 | Servers, NAS, self-hosted services |
| 50 | Management | 192.168.50.0/24 | Infrastructure management: switches, APs, OPNsense itself |

### Creating VLANs in OPNsense

For each VLAN:

1. Go to **Interfaces > Other Types > VLAN**.
2. Create a new VLAN:
   - **Parent interface**: the physical interface connected to the managed switch (for example, `igb1`).
   - **VLAN tag**: the ID from the table above (10, 20, 30, 40, 50).
   - **Description**: the name of the VLAN.
3. Go to **Interfaces > Assignments** and assign each VLAN as a new interface.
4. Configure each interface:
   - Enable the interface.
   - Assign a static IP: the gateway IP for that subnet (for example, `192.168.10.1/24` for VLAN 10).
5. Configure DHCP in **Services > DHCPv4** for each VLAN with its corresponding range.

### Switch configuration

The managed switch needs to be configured to understand VLANs:

- The **trunk** port going to OPNsense must be **tagged** for all VLANs (10, 20, 30, 40, 50).
- **Access ports** are configured as **untagged** in the corresponding VLAN. For example, the port where a guest AP is connected is set to untagged on VLAN 20.
- If the AP supports multiple SSIDs with VLANs (like Ubiquiti or TP-Link Omada), you can create one SSID per VLAN and the AP handles tagging the traffic.

### Firewall rules between VLANs

This is the point where segmentation is really defined. Without firewall rules, VLANs share the same router and can communicate with each other. Explicit rules need to be created.

**General principle**: deny everything between VLANs by default and allow only what's necessary.

**VLAN 10 (Main)**:

| Action | Source | Destination | Port | Description |
|--------|--------|-------------|------|-------------|
| Pass | Main net | * | * | Full internet access |
| Pass | Main net | Servers net | * | Access to internal services |
| Block | Main net | Management net | * | No direct access to management |
| Block | Main net | IoT net | * | IoT isolation |

**VLAN 20 (Guests)**:

| Action | Source | Destination | Port | Description |
|--------|--------|-------------|------|-------------|
| Pass | Guests net | * | 80, 443, 53 | Only web browsing and DNS |
| Block | Guests net | RFC1918 | * | No access to internal networks |

**VLAN 30 (IoT)**:

| Action | Source | Destination | Port | Description |
|--------|--------|-------------|------|-------------|
| Pass | IoT net | * | 443, 8883 | Only HTTPS and MQTT for cloud |
| Pass | IoT net | IoT gateway | 53 | DNS |
| Block | IoT net | RFC1918 | * | No access to internal networks |

**VLAN 40 (Servers)**:

| Action | Source | Destination | Port | Description |
|--------|--------|-------------|------|-------------|
| Pass | Servers net | * | 80, 443, 53 | Internet access for updates |
| Pass | Servers net | Servers net | * | Inter-service communication |
| Block | Servers net | Main net | * | Servers don't initiate connections to Main |

**VLAN 50 (Management)**:

| Action | Source | Destination | Port | Description |
|--------|--------|-------------|------|-------------|
| Pass | Management net | * | * | Full access (admins only) |

To implement the RFC1918 network blocking rule (which covers all private subnets), create an alias in **Firewall > Aliases**:

- Name: `RFC1918`
- Type: Network
- Content: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`

This alias is used as the destination in blocking rules to prevent VLANs like Guests or IoT from accessing any internal network.

## Hardening and best practices

### Updates

The first and most basic thing: keep OPNsense updated. Security updates are published regularly and patches are applied quickly.

- Configure email update notifications.
- Apply updates during low-usage times.
- Before updating, make a backup (it's already automated if you followed the first section).

### DNS over TLS (DoT)

Configure Unbound (OPNsense's DNS resolver) to use DNS over TLS:

In **Services > Unbound DNS > General**:

1. Enable **DNS over TLS**.
2. In **Custom forwarding**, add DNS servers that support DoT:

```
# Quad9 (includes malware filtering)
9.9.9.9@853#dns.quad9.net
149.112.112.112@853#dns.quad9.net

# Cloudflare
1.1.1.1@853#cloudflare-dns.com
1.0.0.1@853#cloudflare-dns.com
```

This encrypts DNS queries between OPNsense and the resolver, preventing the ISP from seeing which domains each device queries.

### Disable unnecessary services

In **System > Settings > Administration**:

- Disable **UPnP** unless strictly necessary (and even then, limit it to specific interfaces).
- Disable **SNMP** if it's not used for monitoring.
- Review installed plugins and uninstall those not in use.

### Centralized logging

OPNsense can send logs to an external syslog server. If you have a monitoring stack (Grafana + Loki, or ELK), configure sending in **System > Settings > Logging > Remote**:

- Server: the syslog server IP.
- Protocol: TCP with TLS if possible.
- Facility: select which logs to send (firewall, system, IDS).

### Regular audit

Establish a review routine:

- **Weekly**: review IDS/IPS and CrowdSec logs. Look for recurring patterns.
- **Monthly**: review firewall rules. Are there rules that no longer make sense? Has any new device been added that needs specific rules?
- **Quarterly**: review users and permissions. Is each user still necessary? Are SSH keys still valid?

In the third and final post of the series, we'll review everything configured, check the overall security posture, and look at advanced practices.