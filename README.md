# 🛡️ SOC Homelab — Wazuh SIEM & Threat Detection

> A hands-on homelab simulating a real enterprise SOC environment, built to develop practical skills in threat detection, log analysis, network segmentation, and incident response using open-source tools.

---

## 📌 Overview

This project documents the design, deployment, and operation of a personal Security Operations Center (SOC) homelab. The environment runs entirely on VirtualBox and uses **Wazuh** as the core SIEM platform, with a segmented network managed by **pfSense**, Active Directory managed endpoints, and attack simulations performed from an isolated Kali Linux segment.

The goal is to replicate real-world SOC workflows: generating attacks, detecting threats, analyzing alerts, capturing network traffic, and tuning detection rules.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      HOST: Windows 11                           │
│                   VirtualBox Hypervisor                         │
└────────┬───────────────────────┬────────────────────────────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐     ┌─────────────────────────────────────────┐
│   ATACANTE      │     │           LAN CORPORATIVA               │
│   Segment       │     │         192.168.56.0/24                 │
│ 192.168.50.0/24 │     │                                         │
│                 │     │  ┌──────────────┐  ┌─────────────────┐  │
│ ┌─────────────┐ │     │  │ Wazuh Server │  │ Windows Server  │  │
│ │ Kali Linux  │ │     │  │ Ubuntu 24.04 │  │   2019 + AD     │  │
│ │ 192.168.50.x│ │     │  │192.168.56.10 │  │ 192.168.56.101  │  │
│ │ - Nmap      │ │     │  │              │  │ midominio.local  │  │
│ │ - Hydra     │ │     │  │ Manager      │  │                 │  │
│ │ - Metasploit│ │     │  │ Indexer      │  └────────┬────────┘  │
│ │ - Wireshark │ │     │  │ Dashboard    │           │ AD        │
│ └──────┬──────┘ │     │  └──────────────┘           │ Domain   │
└────────│────────┘     │                    ┌─────────▼────────┐ │
         │              │                    │  Windows 10 Pro  │ │
         │              │                    │  192.168.10.20   │ │
         │              │                    │  + Sysmon        │ │
         │              │                    │  midominio.local │ │
         ▼              │                    └──────────────────┘ │
┌─────────────────┐     └─────────────────────────────────────────┘
│    pfSense      │
│  2.7.2-RELEASE  │◄── WAN: 10.0.2.15 (NAT/Internet)
│                 │◄── LAN: 192.168.56.200 (Corporativa)
│  + Suricata IDS │◄── ATACANTE: 192.168.50.1 (Aislada)
│                 │
│  Syslog → Wazuh │
│  UDP 514        │
└─────────────────┘
```

---

## 🌐 Network Segmentation

| Segment | Subnet | Interface | Purpose |
|---|---|---|---|
| **LAN Corporativa** | `192.168.56.0/24` | em2 | Wazuh Server, Windows Server AD, Windows 10 |
| **ATACANTE** | `192.168.50.0/24` | em1 (opt1) | Kali Linux — isolated attack segment |
| **WAN** | `10.0.2.15/24` | em0 | Internet access via NAT |

pfSense acts as the perimeter firewall between segments, with Suricata IDS monitoring traffic on the LAN interface. Firewall rules restrict communication between the ATACANTE segment and the corporate LAN, simulating a real network boundary.

---

## 🖥️ Infrastructure

| VM | OS | Role | IP | Key Tools |
|---|---|---|---|---|
| **Wazuh-Server** | Ubuntu Server 24.04 LTS | SIEM (Manager + Indexer + Dashboard) | 192.168.56.10 | Wazuh 4.x all-in-one |
| **pfSense** | FreeBSD (pfSense 2.7.2) | Firewall + IDS + Network Segmentation | 192.168.56.200 | Suricata, syslog |
| **Windows Server 2019** | Windows Server 2019 | Domain Controller (Active Directory) | 192.168.56.101 | AD DS, DNS, GPO, Wazuh Agent |
| **Windows 10 Pro** | Windows 10 Pro | Domain-joined endpoint | 192.168.10.20 | Sysmon, Wazuh Agent |
| **Kali Linux** | Kali Linux | Attack simulation (isolated segment) | 192.168.50.x | Nmap, Hydra, Metasploit, Wireshark |

---

## 🔧 Technologies Used

| Tool | Purpose |
|---|---|
| **Wazuh** | SIEM — log collection, correlation, alerting, threat hunting |
| **Suricata** | Network IDS/IPS on pfSense — ET ruleset |
| **pfSense 2.7.2** | Firewall + network segmentation + syslog forwarding |
| **Active Directory** | Domain `midominio.local` — user/policy management |
| **Sysmon** | Advanced Windows endpoint telemetry (Windows 10) |
| **Wireshark** | Network traffic capture and packet analysis |
| **VirusTotal** | Hash analysis and malware intelligence |
| **VirtualBox** | Hypervisor — multi-adapter network architecture |

---

## ⚔️ Attack Simulations & Detections

### 1. Nmap Port Scan Detection
**Tool:** Nmap from Kali Linux (isolated ATACANTE segment)  
**Target:** pfSense LAN (`192.168.56.200`)  
**Detection:** Suricata — `ET SCAN Possible Nmap User-Agent Observed` (Priority 1)  
**Flow:** Kali → pfSense (Suricata) → syslog UDP 514 → Wazuh → Dashboard

```bash
nmap -A -F 192.168.56.200
```
**Result:** 24+ real-time alerts in Wazuh dashboard.

---

### 2. Brute Force Detection (Windows Endpoints)
**Tool:** Hydra / failed login simulation  
**Target:** Windows Server 2019 (`192.168.56.101`)  
**Detection:** Wazuh rule — Event ID 4625 (Failed logon attempts)  
**Visible in:** Threat Hunting module

---

### 3. pfSense Firewall Log Analysis
**Source:** pfSense filterlog via syslog UDP 514  
**Detection:** Custom Wazuh decoder + rules  
**Coverage:** Blocked packets (level 5), Allowed packets (level 3)

---

### 4. Network Traffic Analysis with Wireshark
**Tool:** Wireshark on Kali Linux  
**Usage:** Packet capture during attack simulations to analyze traffic at the protocol level (TCP handshakes, DNS queries, TLS sessions)  
**Value:** Correlates network-level evidence with Wazuh SIEM alerts

---

### 5. Malware Hash Analysis with VirusTotal
**Tool:** VirusTotal API / web interface  
**Usage:** Hash verification of suspicious files detected during attack simulations  
**Value:** Enriches Wazuh alerts with threat intelligence context

---

## 🏢 Active Directory Configuration

- **Domain:** `midominio.local`
- **Domain Controller:** `WIN-J02ITKGELIK.midominio.local`
- **Organizational Units:** Laboratorio, SOPORTE
- **Windows 10 Pro** joined to domain — managed via GPO
- Wazuh agent monitors both DC and domain-joined endpoint
- Event ID 4625, 4624, 4648 monitored for authentication anomalies

---

## 📋 Wazuh Custom Rules (`local_rules.xml`)

```xml
<group name="local,">

  <!-- pfSense base decoder rule -->
  <rule id="100100" level="0">
    <decoded_as>pfsense</decoded_as>
    <description>pfSense Firewall Logs</description>
  </rule>

  <!-- Blocked packets -->
  <rule id="100101" level="5">
    <if_sid>100100</if_sid>
    <action>block</action>
    <description>pfSense: Blocked packet on Firewall</description>
    <group>firewall,pfsense,packet_drop,</group>
  </rule>

  <!-- Allowed packets -->
  <rule id="100102" level="3">
    <if_sid>100100</if_sid>
    <action>pass</action>
    <description>pfSense: Allowed packet on Firewall</description>
    <group>firewall,pfsense,packet_allow,</group>
  </rule>

  <!-- Suricata IDS alerts via syslog EVE -->
  <rule id="100001" level="8">
    <match>ET SCAN|ET EXPLOIT|ET MALWARE|Priority: 1</match>
    <description>Suricata IDS Alert detected</description>
    <group>suricata,ids,attack,</group>
  </rule>

</group>
```

---

## 🔗 Integrations

### pfSense → Wazuh
- Protocol: UDP syslog port 514
- Remote log server: `192.168.56.10:514`
- Content: **Everything** (all pfSense system events)

### Suricata → Wazuh
- EVE JSON Output Type: **SYSLOG**
- Alerts forwarded via pfSense system log daemon
- Wazuh matches ET signature strings in rule `100001`

### Windows Server 2019 + Windows 10 → Wazuh
- Wazuh agent installed on both endpoints
- Audit policies configured via GUIDs (Spanish-language OS)
- Sysmon on Windows 10 for enhanced endpoint telemetry
- Event IDs monitored: 4624, 4625, 4648, 4672, 4688

---

## 🐛 Technical Challenges Solved

### 1. El Salvador Ubuntu Mirror Failure
`sv.archive.ubuntu.com` is frequently unreliable. Fixed by replacing with the official mirror during and after installation:
```bash
sudo sed -i 's|http://sv.archive.ubuntu.com|http://archive.ubuntu.com|g' \
  /etc/apt/sources.list.d/ubuntu.sources
```

### 2. Suricata Alerts Not Reaching Wazuh Dashboard
**Root cause:** Suricata EVE Output Type was set to `FILE` instead of `SYSLOG`.  
**Fix 1:** Changed to `SYSLOG` in pfSense → Services → Suricata → Interface → EVE Output Settings.  
**Fix 2:** Updated Wazuh rule to match ET signature strings via `<match>` tag instead of `<program_name>`, since pfSense syslog wraps Suricata output differently.

### 3. LVM Disk Only Showing 25GB
Ubuntu LVM allocates ~50% of disk by default. Fixed during installation by manually editing `ubuntu-lv` size from 25GB → 51GB in the storage configurator.

### 4. Windows Audit Policies via GUIDs
Spanish-language Windows Server 2019 required configuring audit policies using GUIDs instead of policy display names to ensure Wazuh agent compatibility.

### 5. Network Segmentation Design
Designed a two-segment network in pfSense to isolate the attack machine (Kali) from the corporate LAN, simulating real DMZ/attacker boundary scenarios. pfSense firewall rules control inter-segment traffic while Suricata monitors all LAN traffic.

---

## 🎯 Skills Demonstrated

- SIEM deployment and configuration (Wazuh all-in-one)
- Network segmentation design (pfSense multi-interface)
- Network IDS integration (Suricata + EVE JSON + syslog)
- Active Directory setup and GPO management
- Endpoint monitoring with Sysmon and Wazuh agents
- Custom decoder and detection rule development (XML)
- Log pipeline troubleshooting (syslog, EVE JSON, archives)
- Attack simulation and detection validation
- Network traffic analysis (Wireshark)
- Threat intelligence enrichment (VirusTotal hash analysis)
- Linux server administration (Ubuntu Server 24.04)
- Windows Server administration (AD DS, DNS, audit policies)

---

## 🗺️ Roadmap

- [ ] MITRE ATT&CK mapping for Suricata and Sysmon rules
- [ ] Splunk integration for SIEM comparison
- [ ] Vulnerability scanning with Wazuh SCA policies
- [ ] Metasploit exploitation scenarios with full kill chain documentation
- [ ] Active Directory attack detection (Pass-the-Hash, Kerberoasting)
- [ ] Automated threat response with Wazuh Active Response

---

## 📚 References

- [Wazuh Documentation](https://documentation.wazuh.com)
- [Suricata Emerging Threats Rules](https://rules.emergingthreats.net)
- [pfSense Documentation](https://docs.netgate.com/pfsense)
- [Sysmon Configuration](https://github.com/SwiftOnSecurity/sysmon-config)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [VirusTotal](https://www.virustotal.com)

---

*Built by a self-directed cybersecurity learner focused on SOC analyst skills. Bilingual (Spanish/English). Based in El Salvador.*
