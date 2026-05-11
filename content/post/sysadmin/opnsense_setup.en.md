+++
author = "Adur"
title = "OPNsense: from hardware to a working firewall"
date = "2026-04-01"
description = "OPNsense installation from hardware selection to having an operational firewall with IDS/IPS, CrowdSec, WireGuard, and well-defined rules."
tags = [
    "opnsense",
    "firewall",
    "ids",
    "ips",
    "crowdsec",
    "wireguard",
    "homelab",
    "networking"
]
categories = [
    "sysadmin"
]
series = ["OPNsense"]
toc = true
+++

## What hardware OPNsense needs

Before opening the installer, it's worth knowing what OPNsense requires and what makes sense to buy. The official documentation distinguishes between minimum and recommended, but in practice there are nuances that matter quite a bit.

### Official minimum requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 64-bit x86-64, 1 GHz | Recent multi-core with AES-NI |
| RAM | 2 GB | 8 GB |
| Storage | 40 GB SSD | 120 GB SSD |
| NICs | 2 network interfaces | 2+ Intel interfaces |

AES-NI has been mandatory since OPNsense 24.1. Without this instruction the installer won't even boot. Any Intel processor from sixth generation onward includes it, and it's been present in AMD since Ryzen.

### Budget options that work

The cheapest option I've had good experience with is a mini PC with an Intel N100 or N200 processor. They're designed for low power consumption and have AES-NI, which covers the main requirement. They can be found with four Intel i226-V Ethernet ports for under 150 euros.

Some specific models that work:

- **Topton N100/N200 with 4x i226-V**: between 120 and 170 euros depending on configuration. They come without RAM or SSD, which are purchased separately. An 8 GB DDR5 module and a 128 GB SSD add about 30 euros more.
- **Protectli VP2420 or VP2410**: more expensive (around 300 euros), but with official support and an aluminum case that dissipates heat well. A good option if you prefer something with serious warranty.
- **Recycled hardware with two NICs**: a Dell OptiPlex or Lenovo ThinkCentre with a dual-port Intel PCIe card works perfectly. They can be found for 50-80 euros in second-hand markets.

What really matters: make sure the network interfaces are Intel. Realtek works, but they cause problems with offloading and performance under load. It's worth paying the difference.

## Installation

Installing OPNsense is straightforward. Download the ISO image from the official website, write it to a USB with `dd` or Rufus on Windows, and boot from the USB.

```bash
# Write the image to a USB (be careful to select the correct device)
dd if=OPNsense-24.7-dvd-amd64.iso of=/dev/sdX bs=4M status=progress
```

On boot, a live environment appears with the option to try before installing. The default user is `installer` with password `opnsense`.

During installation:

1. Select the destination disk for installation (the internal SSD, not the USB).
2. Choose the filesystem. **UFS** is the simple and stable option. **ZFS** has advantages (snapshots, compression), but for a firewall with a single disk, UFS is sufficient.
3. Define the root password.
4. Remove the USB and reboot.

After reboot, OPNsense boots directly and presents a text console with a basic menu. From there you can assign interfaces and configure the LAN IP address to access the web interface.

## PPPoE configuration on WAN

If your ISP uses PPPoE (as is the case with many fiber connections in Spain and Latin America), it needs to be configured on the WAN interface.

In **Interfaces > WAN**:

1. Change the IPv4 configuration type to **PPPoE**.
2. Enter the username and password provided by the ISP.
3. With most providers you don't need to touch the MTU, but if you notice fragmentation issues, adjust it to **1492** (the standard for PPPoE over Ethernet with MTU 1500).
4. Save and apply.

If the connection doesn't come up, verify that the ONT cable goes directly to the port assigned as WAN. Some ISP ONTs need to be in bridge mode for OPNsense to negotiate the PPPoE session directly.

## LAN in bridge mode

There are situations where it's useful to group multiple physical ports into the same network segment, for example when the mini PC has four ports and we want three of them to work as a switch without additional hardware.

To configure a bridge in OPNsense:

1. Go to **Interfaces > Other Types > Bridge**.
2. Create a new bridge and add the interfaces you want to group (for example, `igb1`, `igb2`, `igb3`).
3. Go to **Interfaces > Assignments** and assign the newly created bridge as the LAN interface.
4. Configure the static LAN IP on the bridge interface (for example, `192.168.1.1/24`).
5. Enable the DHCP server in **Services > DHCPv4** pointing to the bridge interface.

With this, all three ports share the same network segment and DHCP serves addresses for all of them.

## Wireless interface and firmware

OPNsense supports some WiFi cards, but the support is limited compared to Linux. Atheros cards work best, but many need additional firmware that isn't included by default.

To install the required firmware:

```bash
# From the OPNsense console (option 8 from the menu for shell)
pkg install wifi-firmware-atheros
# Or for Intel cards:
pkg install wifi-firmware-intel
```

After installing the firmware, restart the system. The wireless interface should appear in **Interfaces > Assignments**.

To configure the access point:

1. Go to **Interfaces > Wireless** and create a new device in **Access Point** mode.
2. Select the standard (802.11ac/ax if the card supports it).
3. Configure the SSID and WPA2/WPA3 security.
4. Assign the wireless interface and give it an IP in a different range from the wired LAN, or add it to the existing bridge if you want it on the same segment.

An honest warning: integrated WiFi in OPNsense works, but don't expect the performance or stability of a dedicated access point. For serious use, it's better to use an external AP (Ubiquiti, TP-Link Omada) and let OPNsense handle only routing and firewall.

## IDS and IPS: intrusion detection and prevention

OPNsense includes Suricata as the IDS/IPS engine. The difference between the two modes is simple: IDS detects and logs, IPS detects and blocks.

### Initial configuration

In **Services > Intrusion Detection**:

1. **Enable IDS**: check the activation box.
2. **IPS mode**: if you want it to block traffic, change the mode to IPS. This requires Suricata to run in inline mode, which is the default behavior in OPNsense.
3. **Interfaces**: select WAN at minimum. If you want to inspect internal traffic as well, add LAN, but this consumes quite a bit more CPU.
4. **Pattern matcher**: select **Hyperscan** if the hardware supports it (processors with SSSE3). It's significantly faster than the default matcher.

### Recommended rule sets in 2026

It's not about activating all available rules. That consumes resources and generates false positives. A reasonable selection:

| Rule set | What it's for | Recommendation |
|----------|---------------|----------------|
| **ET Open (Emerging Threats)** | Known threats, malware, C2 | Activate. It's the foundation |
| **Abuse.ch SSL Blacklist** | Certificates associated with malware | Activate |
| **Abuse.ch URLhaus** | Malware distribution URLs | Activate |
| **ET Open Compromised** | Known compromised IPs | Activate |
| **Feodo Tracker** | Banking botnets | Activate |
| **ET Open Tor** | Tor traffic | Only if you want to block Tor |
| **Snort VRT** | Snort commercial rules | Requires subscription, not essential |

### Best practices for IDS/IPS in 2026

- **Don't activate all rules**. Select the ones that make sense for your environment. A home network doesn't need SCADA or SQL server rules.
- **Automatic rule updates**: configure scheduled rule downloads. In **Schedule** within IDS, set a daily update.
- **Review logs before switching to IPS mode**. Leave the system in IDS mode for at least a week to identify false positives. If something legitimate triggers alerts, create an exception before starting to block.
- **Monitor CPU usage**. Suricata can consume a lot on modest hardware. If the processor stays above 80% usage with IPS active, reduce the number of rules or limit inspection to the WAN interface.
- **Use EVE JSON logging** to export events to a SIEM or analysis tool. The JSON format facilitates integration with Elasticsearch, Grafana, or Wazuh.
- **Don't rely solely on IDS/IPS**. It's one more layer of defense. It doesn't replace good firewall rules, network segmentation, or regular updates.

## CrowdSec

CrowdSec complements Suricata with a different approach: log analysis and shared decisions with the community. While Suricata inspects packets in real time, CrowdSec analyzes service logs and applies bans based on behavior patterns.

### Installation

CrowdSec has an official plugin for OPNsense:

1. Go to **System > Firmware > Plugins**.
2. Search for `os-crowdsec` and install it.
3. After installation, it appears in **Services > CrowdSec**.

### Recommended configuration and collections

(Optional) After installation, register the instance in the CrowdSec central console:

```bash
# From the OPNsense shell
cscli console enroll <your-enrollment-key>
```

The collections define what attack patterns CrowdSec detects. Recommended for a home or small office firewall:

```bash
# Base collection for firewalls (usually already installed)
cscli collections install crowdsecurity/freebsd

# Detection of port scanning and SSH brute force
cscli collections install crowdsecurity/sshd
cscli collections install crowdsecurity/iptables

# HTTP protection if you expose web services
cscli collections install crowdsecurity/nginx
cscli collections install crowdsecurity/base-http-scenarios

# Aggressive scan detection
cscli collections install crowdsecurity/http-cve

# Community blocklists (known malicious IPs)
cscli collections install crowdsecurity/whitelist-good-actors
```

Enable the **firewall bouncer** so CrowdSec can create blocking rules directly in the OPNsense firewall:

```bash
cscli bouncers add opnsense-firewall
```

The generated token is entered in the plugin configuration in the web interface, in **Services > CrowdSec > Bouncer**.

The real advantage of CrowdSec is shared intelligence. When a community member detects an attacking IP, that information is distributed to everyone else. It's like having a collaborative IP reputation system.

## WireGuard

WireGuard is the cleanest option for VPN in 2026. Faster, simpler, and with better cryptography than OpenVPN or IPsec.

### Server configuration

In **VPN > WireGuard**:

1. **Create an instance**: go to the **Instances** tab (or **Local** in earlier versions) and add a new one.
   - Generate a key pair (done automatically when creating the instance).
   - Listen port: `51820` (or whatever you prefer).
   - Tunnel address: `10.10.10.1/24`.

2. **Add a peer**: in the **Peers** tab, create a new pair.
   - Client's public key (generated on the client device).
   - Allowed IPs: `10.10.10.2/32` (the IP the client will have inside the tunnel).

3. **Assign the interface**: go to **Interfaces > Assignments**, assign the WireGuard interface (`wg0` or `wg1`), enable it, and don't touch the IP configuration (it's already defined in WireGuard).

### Client configuration

On the client (mobile, laptop), the configuration is a simple file:

```ini
[Interface]
PrivateKey = <client-private-key>
Address = 10.10.10.2/24
DNS = 10.10.10.1

[Peer]
PublicKey = <server-public-key>
Endpoint = <public-ip-or-ddns>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

With `AllowedIPs = 0.0.0.0/0`, all client traffic goes through the tunnel. If you only want to access the local network, change to `AllowedIPs = 192.168.1.0/24, 10.10.10.0/24`.

## Firewall rules

It's no use configuring services if the firewall rules don't allow the correct traffic. OPNsense blocks everything by default on WAN, which is correct. The work is in allowing what's necessary on LAN and WireGuard.

### Rules for LAN

In **Firewall > Rules > LAN**:

| Action | Protocol | Source | Destination | Destination Port | Description |
|--------|----------|--------|-------------|------------------|-------------|
| Pass | IPv4+6 | LAN net | * | * | Allow LAN outbound |
| Pass | IPv4 | LAN net | LAN address | 53 | DNS to firewall |
| Pass | IPv4 | LAN net | LAN address | 443 | WebUI access |

The first rule is the most permissive and allows LAN to go out to the internet. In the second post of the series we'll see how to restrict this with VLANs.

### Rules for WireGuard

In **Firewall > Rules > WireGuard** (or the assigned interface):

| Action | Protocol | Source | Destination | Destination Port | Description |
|--------|----------|--------|-------------|------------------|-------------|
| Pass | IPv4 | WireGuard net | LAN net | * | LAN access from VPN |
| Pass | IPv4 | WireGuard net | * | * | Internet outbound from VPN |

### Rule on WAN for WireGuard

In **Firewall > Rules > WAN**, add a rule to allow incoming connection to the WireGuard port:

| Action | Protocol | Source | Destination | Destination Port | Description |
|--------|----------|--------|-------------|------------------|-------------|
| Pass | UDP | * | WAN address | 51820 | Incoming WireGuard |

## Offloading and hardware tuning

When everything is working, it's time to squeeze out the performance. OPNsense runs on FreeBSD, and it has several offloading options that can make a noticeable difference.

### What offloading is

Offloading means delegating certain network operations to the network card hardware instead of processing them in software on the CPU. This frees up CPU cycles for other tasks (like Suricata or CrowdSec) and reduces latency.

### Available options

In **Interfaces > Settings**:

- **Hardware CRC (Checksum Offloading)**: delegates TCP/UDP/IP checksum calculation to the network card. Enable if the NIC supports it (Intel i210/i225/i226 do). Measurably reduces CPU load.
- **Hardware TSO (TCP Segmentation Offloading)**: the network card handles splitting large TCP packets into smaller segments. Improves throughput in large transfers. Can cause problems with some IPS configurations, so test and verify.
- **Hardware LRO (Large Receive Offloading)**: groups small incoming packets into larger blocks before passing them to the CPU. Reduces interrupts. **Do not enable if using IPS in inline mode**, as it interferes with packet inspection.
- **VLAN Hardware Filtering**: if VLANs are used (we'll cover this in the second post), let the NIC filter by VLAN ID in hardware.

### Additional system tuning

In **System > Settings > Tunables**, some useful adjustments:

```
# Increase network buffers
net.inet.tcp.recvspace=65536
net.inet.tcp.sendspace=65536

# Enable RACK and BBR if hardware supports it (FreeBSD 14+)
net.inet.tcp.functions_default=bbr

# Adjust the number of NIC queues
hw.igb.num_queues=4
```

Don't obsess over tuning. On most home connections (up to 1 Gbps symmetric), an N100 with the default settings already gives maximum performance. Tuning starts to matter when you want to squeeze 2.5 Gbps connections or more, or when Suricata consumes too much CPU.

## User management and web interface security

Using `root` for day-to-day web interface access is bad practice. If someone compromises those credentials, they have full control.

### Create a new administrator user

In **System > Access > Users**:

1. Create a new user with a descriptive name (not `admin`, something less predictable).
2. Assign a strong password. Minimum 16 characters, generated with a password manager.
3. In **Effective Privileges**, assign the `admins` group.
4. Enable OTP authentication if possible. OPNsense supports native TOTP, so it can be used with any authenticator app.

### Change the root password

In **System > Access > Users**, select `root` and change the password to something long and random. Store this password in a safe place (password manager) and don't use it for daily access.

A more radical option: disable root login on the web interface. This can be done by removing web access privileges from the root user, leaving only physical console access as a last resort.

### Restrict web interface access

By default, the web interface is accessible from the entire LAN. This can be restricted:

1. **Change the HTTPS port**: in **System > Settings > Administration**, change the port from `443` to another non-standard one (for example, `8443`).
2. **Restrict by source IP**: create an alias in **Firewall > Aliases** with the IPs from which web interface access is allowed. Then, in the LAN firewall rules, create a rule that allows access to the WebUI port only from that alias, and a blocking rule for everything else.
3. **Enable HTTPS with your own certificate**: in **System > Trust > Certificates**, generate a self-signed certificate or import one from Let's Encrypt. This eliminates browser warnings and ensures the connection is properly encrypted.
4. **Brute force protection**: in **System > Settings > Administration**, configure the maximum number of failed attempts and the lockout time.
5. **Disable HTTP**: make sure only HTTPS is enabled. Unencrypted HTTP access to a firewall's admin interface makes no sense.

In the next post in the series, we'll take security further with encrypted backups, VLANs, Zenarmor, and system hardening.