+++
author = "Adur"
title = "OPNsense: Auditing, Automation, and Advanced Practices"
date = "2026-04-11"
description = "Complete security posture review, Infrastructure as Code with the API and Ansible, auditing framework based on ISO 27001 and ENS, and comparison with VyOS and OpenWrt to decide the future of your network."
tags = [
    "opnsense",
    "firewall",
    "iac",
    "ansible",
    "iso27001",
    "ens",
    "audit",
    "homelab",
    "networking"
]
categories = [
    "sysadmin"
]
series = ["OPNsense"]
toc = true
+++

## Where We Stand

In the first two posts of the series, we set up OPNsense from scratch and took it to a configuration that is no longer trivial. Let's do a quick recap before continuing.

| Layer | What Was Configured | Post |
|-------|---------------------|------|
| Hardware | Mini PC with Intel N100/N200, Intel i226-V NICs | First |
| Connectivity | PPPoE on WAN, bridge on LAN, wireless interface | First |
| Detection | Suricata IDS/IPS with ET Open, Abuse.ch, Feodo rulesets | First |
| Shared Intelligence | CrowdSec with firewall bouncer and community collections | First |
| VPN | WireGuard with configured peers | First |
| Firewall Rules | Basic rules for LAN, WireGuard, and WAN | First |
| Offloading | Checksum, TSO, TCP buffer tuning | First |
| Users | Dedicated admin with OTP, restricted root, protected WebUI | First |
| Backups | AES-256-CBC encrypted, automatic, external storage | Second |
| DPI | Zenarmor with per-interface and category policies | Second |
| Segmentation | 5 VLANs (Main, Guests, IoT, Servers, Management) with inter-VLAN rules | Second |
| SSH | ed25519 keys only, non-standard port, IP-restricted access | Second |
| DNS | Unbound with DNS over TLS to Quad9 and Cloudflare | Second |
| Logging | Remote syslog with TCP/TLS to Grafana+Loki or ELK | Second |

It's a solid configuration. But so far everything has been done manually, through the web interface, without a formal review process or a way to reproduce the configuration if something breaks beyond the XML backup. This post covers what's missing: advanced practices, automation, a serious auditing framework, and a comparison with alternatives to make informed decisions.

## Security Posture Review

### Defense in Depth: What We Have

If we look at what we've configured as defense layers, the architecture has some depth:

| Defense Layer | OPNsense Component | Current State |
|---------------|--------------------|---------------|
| Perimeter | WAN deny-all rules + incoming WireGuard | Functional |
| Intrusion Detection | Suricata IPS with ET Open and Abuse.ch | Functional, default tuning |
| Log Analysis | CrowdSec bouncer + community intelligence | Functional |
| Application Inspection | Zenarmor DPI with per-VLAN policies | Functional |
| Segmentation | 5 VLANs with deny-all inter-VLAN rules by default | Functional |
| Access Control | Dedicated admin, OTP, SSH with ed25519 keys | Functional |
| DNS Encryption | Unbound with DoT to Quad9/Cloudflare | Functional |
| Backups | Encrypted, automatic, external storage | Functional |

Five layers of detection and prevention, real segmentation, and reasonable access control. For a homelab or small office, it's more than most have. But there are gaps.

### Remaining Gaps

Being honest, there are several things that haven't been addressed that matter:

- **No geolocation filtering**. The entire planet can try to connect to exposed ports on WAN. Most automated attacks come from IP ranges with which you have no legitimate relationship.
- **No reverse proxy**. If self-hosted services are exposed (Nextcloud, Jellyfin), they go directly via NAT without TLS termination or application-level protection.
- **Suricata is at default configuration**. Rules are enabled, but performance parameters haven't been tuned and irrelevant categories haven't been removed.
- **Everything has been configured manually**. If tomorrow OPNsense needs to be rebuilt from scratch, the XML backup is the only option. There are no playbooks, no version control of the configuration, no way to review what changed and when without opening the web interface history.
- **No formal audit cadence**. The second post mentioned a weekly/monthly/quarterly routine, but without a framework behind it, it's easy for it to remain good intentions.
- **No automated threat feeds** beyond Suricata rule updates. IP blocklists don't update themselves.

The following sections cover these gaps.

## Advanced Practices

### GeoIP Blocking with MaxMind

The idea is simple: if there's no legitimate reason for a connection to come from certain countries, blocking those IP ranges reduces noise. It's not a real security measure, because any attacker with a VPN bypasses it, but it does eliminate a significant amount of automated scans and brute force attacks.

OPNsense uses MaxMind's free GeoLite2 databases, which require an account.

1. Register a free account at MaxMind and generate a license key.
2. In **Firewall > Aliases > GeoIP settings**, enter the license key.
3. Create a GeoIP type alias in **Firewall > Aliases**:
   - Name: `GeoIP_Block`
   - Type: GeoIP
   - Content: select the countries to block.
4. In **Firewall > Rules > WAN**, add a rule at the top:
   - Action: Block
   - Source: `GeoIP_Block`
   - Destination: *
   - Description: Geographic blocking

One warning: keep this list updated. MaxMind updates the databases weekly. Configure automatic update in the GeoIP settings so they don't become obsolete.

### Advanced Suricata Tuning

OPNSense 26.1 includes Suricata 8, which improves multi-thread performance and adds support for new protocols. But the default configuration is conservative. If the hardware has headroom, tuning these parameters makes a difference.

In **Services > Intrusion Detection > Administration**, the advanced section allows configuring parameters not in the standard interface. The most relevant:

```yaml
# /usr/local/opnsense/service/conf/actions.d/conf.d/
# Adjust based on available RAM (these values are for 8 GB)

# Maximum memory for flow tracking
flow:
  memcap: 256mb
  hash-size: 65536

# Maximum memory for TCP stream reconstruction
stream:
  memcap: 512mb
  reassembly.memcap: 256mb

# Ring buffer size for af-packet
af-packet:
  - interface: igb0
    ring-size: 30000
    cluster-type: cluster_flow
```

Beyond memory tuning, review the active rule categories. If there are no SCADA servers, exposed databases, or SMTP services on the network, disable those categories. Each active rule consumes CPU on every inspected packet.

To identify the rules generating the most noise, analyze the EVE JSON log:

```bash
# Top 10 alerts by SID in the last 24 hours
cat /var/log/suricata/eve.json | \
  jq -r 'select(.event_type=="alert") | .alert.signature_id' | \
  sort | uniq -c | sort -rn | head -10
```

If a rule generates hundreds of daily alerts with none being true positives, disable it or adjust its threshold.

### Reverse Proxy with HAProxy and ACME

If self-hosted services are exposed to the internet, doing it directly with port forwarding is functional but insecure. A reverse proxy allows terminating TLS with valid certificates, applying rate limiting, and having a centralized control point.

Install the `os-haproxy` and `os-acme-client` plugins from **System > Firmware > Plugins**.

**Certificates with Let's Encrypt (ACME)**:

1. In **Services > ACME Client > Accounts**, create an ACME account.
2. In **Services > ACME Client > Challenge Types**, configure a DNS-01 challenge. It's preferred over HTTP-01 because it doesn't require opening port 80 and supports wildcards.
3. In **Services > ACME Client > Certificates**, create certificates for each service.

**HAProxy Configuration**:

1. In **Services > HAProxy > Real Servers**, create a backend for each internal service (Nextcloud at `192.168.40.10:443`, Jellyfin at `192.168.40.11:8096`, etc.).
2. In **Services > HAProxy > Rules & Checks > Conditions**, create conditions based on SNI or hostname.
3. In **Services > HAProxy > Virtual Services > Public Services**, create a frontend that listens on port 443, binds the ACME certificates, and routes to backends based on conditions.
4. Enable HSTS in HTTP headers to force HTTPS.
5. Configure rate limiting per IP to mitigate brute force attacks against login forms.

### Threat Feeds and Scheduled Aliases

IP blocklists lose value if they aren't updated. OPNsense allows creating URL Table type aliases that are downloaded automatically.

In **Firewall > Aliases**, create aliases with these sources:

| Alias | URL | Frequency | Description |
|-------|-----|------------|-------------|
| `Spamhaus_DROP` | `https://www.spamhaus.org/drop/drop.txt` | Daily | Hijacked ranges or used for spam |
| `Spamhaus_EDROP` | `https://www.spamhaus.org/drop/edrop.txt` | Daily | Extension of DROP |
| `Abusech_Feodo` | `https://feodotracker.abuse.ch/downloads/ipblocklist.txt` | Every 6h | Banking botnet IPs |
| `Abusech_SSLBL` | `https://sslbl.abuse.ch/blacklist/sslipblacklist.txt` | Every 6h | IPs with malicious certificates |

Apply these aliases as source in blocking rules on WAN. URL Table type aliases update automatically according to the configured frequency.

### ZFS Snapshots for Rollback

If ZFS was chosen as the filesystem during installation (instead of UFS), one of its best features can be used: instant snapshots with rollback.

Before any important change (firmware update, massive rule change, plugin installation), create a snapshot:

```bash
# Create a snapshot before updating
zfs snapshot zroot/ROOT/default@pre-update-$(date +%Y%m%d)

# List existing snapshots
zfs list -t snapshot

# If something goes wrong, rollback to the previous snapshot
zfs rollback zroot/ROOT/default@pre-update-20260413
```

This returns the entire filesystem to the snapshot state in seconds. It's insurance against failed updates that complements XML configuration backups. The snapshot recovers everything; the XML backup only recovers the configuration.

## Infrastructure as Code for OPNsense

Doing everything from the web interface works, but it has known problems: there's no readable change history, there's no way to review what was modified before applying it, and rebuilding the configuration requires following a step-by-step guide or restoring an opaque backup.

### OPNsense REST API

OPNsense exposes a fairly complete REST API. The documentation is in `/api/` and covers most web interface functionalities: aliases, firewall rules, interface configuration, IDS, CrowdSec, Unbound, HAProxy, and more.

To use the API, create a key/secret pair in **System > Access > Users**, select the administrator user, and generate an API key. OPNsense generates a file with the key and secret.

```bash
# Query firewall aliases
curl -k -u "API_KEY:API_SECRET" \
  https://192.168.1.1/api/firewall/alias/searchItem

# Create a new alias
curl -k -u "API_KEY:API_SECRET" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"alias":{"name":"test_alias","type":"host","content":"10.0.0.1"}}' \
  https://192.168.1.1/api/firewall/alias/addItem

# Apply pending changes to the firewall
curl -k -u "API_KEY:API_SECRET" \
  -X POST \
  https://192.168.1.1/api/firewall/alias/reconfigure
```

The API doesn't cover 100% of the web interface. Some plugin functions (like parts of Zenarmor) aren't exposed. But for firewall management, aliases, rules, interfaces, and most core services, it's sufficient.

### Ansible: ansibleguy/collection_opnsense

The `ansibleguy.opnsense` collection is the most mature option for managing OPNsense as code in 2026. It wraps the REST API in idempotent Ansible modules.

```bash
# Install the collection
ansible-galaxy collection install ansibleguy.opnsense
```

An example playbook managing aliases and firewall rules:

```yaml
---
- name: Configure OPNsense firewall
  hosts: opnsense
  connection: httpapi
  vars:
    ansible_httpapi_port: 443
    ansible_httpapi_use_ssl: true
    ansible_httpapi_validate_certs: false
  tasks:
    - name: Create RFC1918 alias
      ansibleguy.opnsense.alias:
        name: RFC1918
        type: network
        content:
          - "10.0.0.0/8"
          - "172.16.0.0/12"
          - "192.168.0.0/16"
        description: "RFC1918 private networks"

    - name: Create admin alias
      ansibleguy.opnsense.alias:
        name: Admin_IPs
        type: host
        content:
          - "192.168.50.10"
          - "192.168.50.11"
        description: "Admin IPs"

    - name: Block IoT to internal networks
      ansibleguy.opnsense.rule:
        interface: "opt3"  # VLAN 30 IoT
        action: block
        source_net: "IoT net"
        destination_net: "RFC1918"
        description: "IoT without access to internal networks"
```

Ansible supports check mode (`--check`) to see what would change without applying anything. This is especially useful for reviewing firewall changes before executing them, something the web interface doesn't allow.

The main limitation is that not all OPNsense modules are covered by the collection. Services like Zenarmor, CrowdSec, or advanced Suricata configurations may require direct API calls using Ansible's `uri` module.

### Version Control of config.xml

The most straightforward approach to have configuration under version control: export the `config.xml`, encrypt it, and store it in a private Git repository. It's what's already done with the backups from the second post, but integrated into a Git workflow.

```bash
#!/bin/bash
# export_config.sh - Run from cron or manually before changes
BACKUP_DIR="/root/config-backups"
REPO_DIR="/root/opnsense-config"
DATE=$(date +%Y%m%d_%H%M)

# Export current configuration
cp /conf/config.xml "${BACKUP_DIR}/config_${DATE}.xml"

# Encrypt with age (simpler than OpenSSL for this use)
age -r age1publickey... \
  -o "${REPO_DIR}/config_${DATE}.xml.age" \
  "${BACKUP_DIR}/config_${DATE}.xml"

# Clean up unencrypted file
rm "${BACKUP_DIR}/config_${DATE}.xml"

# Commit to repository
cd "${REPO_DIR}"
git add .
git commit -m "config: backup ${DATE}"
git push origin main
```

With this, you have a change history of the configuration with timestamps and the ability to do `git diff` between encrypted versions (or decrypt two versions and compare them with `diff`). It's not as clean as the VyOS model where configuration is plain text, but it works.

### CI/CD Pipeline for Validation

Taking automation one step further: validate configuration changes before applying them. A lightweight pipeline in GitLab CI or GitHub Actions that:

1. Decrypts the `config.xml` in an ephemeral environment.
2. Validates the XML structure with `xmllint`.
3. Checks anti-patterns with custom rules (rules with `source=any destination=any action=pass`, users without OTP, unnecessary services enabled).
4. Runs the Ansible playbook in check mode against a staging environment (if it exists) or just validates the syntax.

This doesn't replace manual testing, but it catches obvious errors before they reach production.

In terms of IaC maturity, OPNsense is at an intermediate point:

| Aspect | OPNsense | VyOS | OpenWrt |
|--------|----------|------|---------|
| REST API | Complete | Complete | Limited (ubus/JSON-RPC) |
| Terraform | No mature provider | Official provider (Foltik/vyos) | No provider |
| Ansible | ansibleguy.opnsense | vyos.vyos (official) | Community roles |
| Config format | XML (config.xml) | Plain text CLI | UCI (plain text) |
| Git workflow | Manual or scripted export | Native (text config) | Manual export |

VyOS wins at IaC by design: its configuration is plain text from day one, with atomic commits and native rollback. OPNsense compensates with a solid REST API and the Ansible collection, but requires more effort to reach the same level of automation.

## Auditing Framework: ISO 27001 and ENS

### Why It Matters Even for a Homelab

It's tempting to think that a formal auditing framework is only for companies. But the reality is more pragmatic: an auditing framework is a checklist that people with more experience than you have already validated. Following it avoids the "I forgot to review X" problem that appears when the security routine depends only on memory.

There's an additional reason if you're in Spain: the National Security Scheme (ENS, Real Decreto 311/2022) is mandatory for the public sector and its technology providers[^2]. This increasingly includes freelancers and small companies providing services to public administrations. Having the infrastructure aligned with the ENS, even at a basic level, isn't just good practice—it's a potential requirement.

### Relevant ISO 27001:2022 Controls

ISO 27001:2022 organizes security controls in Annex A[^3]. The ones that apply directly to what was configured in this series:

| Control | Description | OPNsense Implementation |
|---------|-------------|--------------------------|
| A.8.20 | Networks security | WAN deny-all firewall, IPS, CrowdSec, GeoIP |
| A.8.22 | Networks segregation | 5 VLANs with restrictive inter-VLAN rules |
| A.8.23 | Web filtering | Zenarmor DPI with per-category policies |
| A.8.15 | Logging | Remote syslog with TLS, Suricata EVE JSON |
| A.8.16 | Monitoring activities | Suricata alerts, CrowdSec dashboards, Zenarmor |
| A.8.17 | Clock synchronization | NTP configured in System > General (essential for log correlation) |
| A.5.15 | Access control | Least-privilege policy, dedicated admin |
| A.5.17 | Authentication information | 16+ character passwords, OTP enabled |
| A.5.18 | Access rights | Periodic review of users and permissions |
| A.8.2 | Privileged access | Restricted root, separate admin, SSH keys only |

The standard doesn't prescribe exact review frequencies, but auditors expect: firewall rule review at least quarterly, privileged access review quarterly, and continuous log monitoring with weekly manual review.

### Applicable ENS Measures

The ENS (RD 311/2022) classifies systems into three levels: basic, medium, and high. The relevant measures for a firewall:

| ENS Measure | Description | Minimum Level | Implementation |
|-------------|-------------|---------------|----------------|
| mp.com.1 | Secure perimeter | Medium | WAN firewall, GeoIP, deny-all by default |
| mp.com.2 | Confidentiality protection | Medium | WireGuard VPN, DoT for DNS |
| mp.com.4 | Networks segregation | Medium | VLANs with inter-VLAN rules |
| op.exp.2 | Security configuration | Basic | SSH hardening, disabled services |
| op.exp.8 | Activity logging | Basic | Remote syslog (2-year retention for medium/high) |
| op.acc.1 | Identification | Basic | Unique users, no shared accounts |
| op.acc.5 | Authentication mechanism | Basic | Strong passwords; OTP for medium/high |
| op.acc.7 | Remote access | Basic | WireGuard VPN mandatory for remote admin |
| op.mon.1 | Intrusion detection | Medium | Suricata IPS + CrowdSec |

The CCN-STIC guides from the National Cryptology Center provide detailed implementation instructions. Specifically, CCN-STIC-811 for system interconnection and CCN-STIC-408 for perimeter security are the most relevant for this context[^1].

An important ENS detail: log retention for medium and high levels is a minimum of two years. If the remote syslog doesn't have capacity for that, it needs to be planned. With Loki and compression, firewall logs from a homelab don't take up much space, but it needs to be considered from the design phase.

### Audit Calendar

Combining ISO 27001 and ENS recommendations with what's realistic for a small environment:

| Frequency | Task | Type |
|-----------|------|------|
| Daily | Automated review of IDS/IPS alerts and CrowdSec decisions | Automated |
| Weekly | Manual log review: blocked events, failed login attempts, anomalous traffic | Manual |
| Monthly | Firewall rule review and alias update. Verify threat feeds are updating | Manual |
| Quarterly | Complete rule review: identify rules with no hits, overly permissive rules, forgotten temporary rules | Manual |
| Quarterly | Access review: active users, permissions, valid SSH keys, API tokens | Manual |
| Semi-annual | Basic exposure test: `nmap` from outside the network against the public IP | Manual |
| Annual | Complete security posture review. Compare with previous year's state | Manual |

For automated daily review, a basic script that runs with cron:

```bash
#!/bin/bash
# /root/scripts/daily_audit.sh
# Run with: crontab -e -> 0 7 * * * /root/scripts/daily_audit.sh

LOG="/var/log/daily_audit_$(date +%Y%m%d).log"

echo "=== Daily audit $(date) ===" > "$LOG"

# Critical Suricata alerts in the last 24h
echo -e "\n--- Suricata alerts (severity 1) ---" >> "$LOG"
cat /var/log/suricata/eve.json | \
  jq -r 'select(.event_type=="alert" and .alert.severity==1) | 
  "\(.timestamp) \(.alert.signature) src=\(.src_ip) dst=\(.dest_ip)"' | \
  tail -50 >> "$LOG"

# Active CrowdSec decisions
echo -e "\n--- CrowdSec active decisions ---" >> "$LOG"
cscli decisions list -o raw 2>/dev/null | wc -l >> "$LOG"

# Age of last backup
echo -e "\n--- Last backup ---" >> "$LOG"
LAST_BACKUP=$(ls -t /conf/backup/*.xml 2>/dev/null | head -1)
if [ -n "$LAST_BACKUP" ]; then
  stat -f "%Sm" "$LAST_BACKUP" >> "$LOG"
else
  echo "No backups found" >> "$LOG"
fi

# System status
echo -e "\n--- System resources ---" >> "$LOG"
echo "CPU: $(sysctl -n dev.cpu.0.temperature 2>/dev/null || echo 'N/A')" >> "$LOG"
echo "RAM: $(sysctl -n vm.stats.vm.v_free_count)" >> "$LOG"
echo "States: $(pfctl -si 2>/dev/null | grep 'current entries')" >> "$LOG"

# Send via email or copy to syslog
logger -t daily_audit "Audit completed. See $LOG"
```

This script isn't meant to be a SIEM. It's a first line of automated review that flags if there's something requiring attention. For serious log analysis, the Grafana + Loki combination or a dedicated Wazuh is appropriate.

## OPNsense vs. Alternatives

Before continuing to invest time in OPNsense, it's worth comparing with the other serious open source firewall/router options. Not to migrate now, but to know what's out there and make informed decisions if needs change.

### VyOS

VyOS is a network operating system based on Debian with a CLI configuration model inspired by JunOS. It has no web interface.

What it does better than OPNsense:

- **Plain text configuration**. VyOS config is a readable text file, with atomic commits and native rollback. It's the IaC dream: `git diff` works directly on the configuration.
- **Official Terraform provider** (Foltik/vyos). Real declarative network infrastructure.
- **Advanced routing**. BGP, OSPF, IS-IS at scale. Supports routing tables with more than one million BGP prefixes.
- **Performance**. The VPP dataplane enables 10 Gbps+ throughput with appropriate hardware.
- **Cloud-native**. Direct deployment on AWS, Azure, and GCP with official support.

What it does worse:

- **No GUI**. Everything is CLI or API. The learning curve is steep if you don't come from enterprise networking.
- **Limited integrated security**. There's no equivalent to Suricata with GUI, no DPI like Zenarmor, no integrated CrowdSec. Suricata can be installed manually, but without the integration OPNsense offers.
- **Paid LTS**. Since 2024, stable LTS images require a subscription. Rolling releases are free but without stability guarantee.

### OpenWrt

OpenWrt is a Linux-based router operating system. Its strength is embedded hardware support.

What it does better than OPNsense:

- **Massive hardware support**. Runs on more than 500 commercial router models, plus x86.
- **Native WiFi**. Directly manages wireless interfaces, with excellent driver support and advanced AP configuration.
- **Lightweight**. Can run on 128 MB RAM and 16 MB flash on embedded hardware.
- **Package ecosystem**. More than 27,000 packages available.

What it does worse:

- **Basic security**. nftables firewall without integrated IDS/IPS. Suricata and Snort packages are community-maintained, poorly maintained, and have performance problems on limited hardware.
- **No DPI**. There's no equivalent to Zenarmor.
- **Limited IaC**. UCI is scriptable but there's no Terraform provider or mature REST API.
- **Complicated updates**. On x86, updating OpenWrt means reinstalling and restoring configuration. OPNsense updates with one click.

### When to Choose Each

| Criterion | OPNsense | VyOS | OpenWrt |
|-----------|----------|------|---------|
| Integrated security | Excellent (Suricata, CrowdSec, Zenarmor) | Basic | Minimal |
| IaC and automation | Good (API + Ansible) | Excellent (Terraform + text config) | Limited (UCI) |
| 10G+ performance | Moderate | Excellent (VPP) | Not applicable |
| Learning curve | Moderate (GUI) | Steep (CLI) | Moderate |
| Cost | Free | Paid LTS, free rolling | Free |
| Ideal use case | Perimeter firewall/UTM | Enterprise or cloud edge router | WiFi AP, lightweight gateway |

For a homelab or small office where security is the priority, OPNsense remains the best option. The combination of Suricata, CrowdSec, and Zenarmor with a usable web interface has no equivalent among the alternatives.

VyOS makes sense if the environment grows toward complex routing (multiple uplinks with BGP, SD-WAN) or if infrastructure is managed exclusively with Terraform.

OpenWrt makes sense as a complement: a WiFi AP running OpenWrt behind an OPNsense is a solid combination. But as a perimeter firewall for security, it falls short.

## Conclusions and Next Steps

In three posts we've gone from a mini PC without an operating system to a segmented network with five VLANs, three detection layers (Suricata, CrowdSec, Zenarmor), VPN with WireGuard, encrypted DNS, automatic backups, automation with Ansible, and an auditing framework based on real standards.

It's not perfect. OPNsense's IaC doesn't reach VyOS's level. Zenarmor has a licensing model that limits advanced features in the free version. And maintaining an audit routine requires discipline that's easy to set aside when everything seems to work well.

But it's a solid foundation to keep building on. There are topics that were deliberately left out of this series because they deserve their own depth:

- **Wazuh SIEM integration**: correlation of Suricata, CrowdSec, and system logs in one place, with centralized alerts and dashboards. It's the natural step for anyone wanting real security monitoring.
- **Multi-WAN and failover**: configure two internet connections with load balancing and automatic failover. Relevant when availability matters.
- **Advanced HAProxy**: mutual TLS (mTLS) with client certificates, certificate authentication for internal services, OAuth2 as an authentication layer against self-hosted applications.
- **Network monitoring with NetFlow/Insight**: detailed traffic analysis by protocol, host, and port to detect anomalies that IDS doesn't catch.
- **Complete automation with Terraform**: if OPNsense gets a mature Terraform provider (or if there's a partial migration to VyOS for routing), declarative management of the entire network.

If there's interest, a fourth post can cover Wazuh integration and advanced monitoring. That's where the jump from "secure homelab" to "infrastructure with real visibility" becomes most evident.

[^1]: The CCN-STIC guides from the National Cryptology Center provide detailed instructions for ENS implementation. Specifically, CCN-STIC-811 covers system interconnection and CCN-STIC-408 covers perimeter security.
[^2]: Real Decreto 311/2022, de 3 de mayo, por el que se regula el Esquema Nacional de Seguridad. BOE-A-2022-7191.
[^3]: ISO/IEC 27001:2022 — Information security, cybersecurity and privacy protection — Information security management systems — Requirements.