# 🔴 Incident Report — IR-2026-001
## Brute Force Attack & Anonymous SMB Access on Domain Controller

**Classification:** HIGH  
**Status:** Contained (Lab Environment)  
**Date:** July 13, 2026  
**Analyst:** Rafael (SOC Analyst — midominio.local)  
**SIEM:** Wazuh 4.x  

---

## 1. Executive Summary

On July 13, 2026, Wazuh SIEM detected a brute force attack originating from an internal host (`192.168.56.102`) targeting the Active Directory Domain Controller (`WIN-J02ITKGELIK`, `192.168.56.101`). The attack generated **152 authentication failures** within a short time window, followed by **anonymous NTLM logon activity**, indicating a potential pass-the-hash or null session attempt.

The attack was detected in real time through Windows Security Event monitoring via the Wazuh agent deployed on the Domain Controller.

---

## 2. Timeline of Events

| Time (CST) | Event | Source |
|---|---|---|
| 19:40 | Attacker begins Nmap port scan against DC | Suricata / Wazuh |
| 19:51 | Hydra brute force initiated via SMB2 (port 445) | Wazuh Agent (Event ID 4625) |
| 19:51 | **152 authentication failures** recorded in 60 seconds | Wazuh — rule 60122 |
| 19:51 | Anonymous NTLM logon detected (possible pass-the-hash) | Wazuh — rule 92652 |
| 19:52 | Attack stopped — session file saved by attacker | Hydra log |
| 19:55 | SOC analyst reviews alerts in Threat Hunting module | Wazuh Dashboard |

---

## 3. Attack Phases (Kill Chain)

### Phase 1 — Reconnaissance
**Tool:** Nmap  
**Command:** `nmap -sn 192.168.56.0/24`  
**Result:** 6 live hosts identified including DC at `192.168.56.101`  
**MITRE ATT&CK:** T1046 — Network Service Discovery  

### Phase 2 — Port Scanning
**Tool:** Nmap  
**Command:** `nmap -sV -p 22,80,135,139,443,445,3389 192.168.56.101`  
**Result:**
- Port 135 — MSRPC (open)
- Port 139 — NetBIOS-SSN (open)  
- Port 445 — Microsoft-DS / SMB (open) ← attack vector
- Port 3389 — RDP (closed)
- OS: Windows Server detected

**MITRE ATT&CK:** T1046 — Network Service Scanning  

### Phase 3 — SMB Enumeration
**Tool:** smbclient  
**Command:** `smbclient -L //192.168.56.101 -N`  
**Result:** Anonymous login successful — null session accepted  
**Finding:** ⚠️ SMB anonymous access enabled on Domain Controller  
**MITRE ATT&CK:** T1135 — Network Share Discovery  

### Phase 4 — Brute Force
**Tool:** Hydra  
**Command:** `hydra -l Administrator -P rockyou.txt 192.168.56.101 smb2 -t 1`  
**Wordlist:** rockyou.txt (14,344,399 passwords)  
**Duration:** ~60 seconds (152 attempts before stopped)  
**Result:** No valid credentials found  
**MITRE ATT&CK:** T1110.001 — Brute Force: Password Guessing  

---

## 4. Indicators of Compromise (IOCs)

| Type | Value | Description |
|---|---|---|
| IP | `192.168.56.102` | Attacker source (Kali Linux) |
| IP | `192.168.56.101` | Target — Domain Controller |
| User | `Administrator` | Targeted account |
| Port | `445/tcp` | SMB attack vector |
| Tool | Hydra v9.6 | Brute force tool detected |
| Tool | Nmap 7.98 | Reconnaissance tool |
| Event ID | 4625 | Failed logon — 152 occurrences |
| Event ID | 4624 | Successful logon (anonymous NTLM) |

---

## 5. Wazuh Alerts Generated

| Rule ID | Level | Description | Count |
|---|---|---|---|
| 60122 | 5 | Logon Failure — Unknown user or bad password | 152 |
| 92652 | 6 | Successful Remote Logon — ANONYMOUS LOGON (possible pass-the-hash) | Multiple |
| 60137 | 3 | Windows User Logoff | Multiple |
| 60106 | 3 | Windows Logon Success | Multiple |

**Total events in 1-hour window:** 1,236  
**Critical alerts (Level 12+):** 2  
**Authentication failures:** 152  

---

## 6. MITRE ATT&CK Mapping

| Tactic | Technique | ID | Tool Used |
|---|---|---|---|
| Reconnaissance | Network Service Discovery | T1046 | Nmap |
| Discovery | Network Share Discovery | T1135 | smbclient |
| Credential Access | Brute Force: Password Guessing | T1110.001 | Hydra |
| Lateral Movement | Use of Alternate Authentication Material | T1550 | NTLM Anonymous |

---

## 7. Root Cause Analysis

**Primary vector:** Port 445 (SMB) accessible from untrusted network segment.

**Contributing factors:**
1. SMB anonymous (null session) authentication enabled on Domain Controller
2. No account lockout policy configured — allowed 152+ login attempts without lockout
3. Administrator account directly targeted (default admin account not renamed/disabled)
4. No network-level block between ATACANTE segment and LAN Corporativa for SMB traffic

---

## 8. Recommendations

### Immediate (Critical)
- [ ] **Disable SMB null sessions** on Domain Controller via GPO:  
  `Network access: Do not allow anonymous enumeration of SAM accounts and shares`
- [ ] **Enable account lockout policy** — recommended: 5 failed attempts → 30 min lockout
- [ ] **Block SMB (port 445) at pfSense** between ATACANTE and LAN segments
- [ ] **Rename or disable built-in Administrator account**

### Short-term (High)
- [ ] Enable **SMB signing** to prevent pass-the-hash attacks
- [ ] Configure **Wazuh Active Response** to auto-block IPs after threshold of failed logins
- [ ] Implement **network segmentation rule** in pfSense to block direct access from endpoints to DC on SMB

### Long-term (Medium)
- [ ] Deploy **Microsoft LAPS** for local administrator password management
- [ ] Enable **Privileged Identity Management** for admin accounts
- [ ] Conduct regular **brute force simulation exercises** and tune Wazuh detection thresholds

---

## 9. Detection Effectiveness

| Detection Method | Result |
|---|---|
| Wazuh Agent (Windows Events) | ✅ Detected in real time |
| Suricata IDS (Network) | ✅ Nmap scan detected |
| Alert threshold (Level 12+) | ✅ 2 critical alerts generated |
| Dashboard visibility | ✅ Attack spike visible in timeline |

**Time to detect:** < 1 minute from attack start  
**Time to investigate:** ~10 minutes  

---

## 10. Lessons Learned

1. **Anonymous SMB access on a Domain Controller is a critical misconfiguration** — any host on the network can enumerate domain information without credentials.

2. **No account lockout policy makes brute force trivial** — 152 attempts in 60 seconds against Administrator should trigger automatic lockout.

3. **Wazuh detection worked correctly** — the combination of Windows agent + audit policies generated actionable alerts in real time, demonstrating the value of endpoint monitoring on critical assets.

4. **Network segmentation worked as designed** — the ATACANTE segment (`192.168.50.x`) was correctly isolated from the LAN Corporativa. However, this attack originated from within the LAN itself (`192.168.56.102`), demonstrating that perimeter controls alone are insufficient against insider threats or compromised internal hosts. Defense in depth — endpoint monitoring via Wazuh agent — was the key detection layer in this scenario.

---

## 11. Response Actions Taken

| Action | Details | Time (CST) |
|---|---|---|
| IP Blocked | `192.168.56.102` blocked via pfSense LAN firewall rule — Action: Block, Protocol: Any, Destination: Any | 20:10 |
| Rule documented | pfSense rule description: "IR-2026-001 - Blocked attacker IP - Brute force against DC" | 20:10 |
| Evidence preserved | `/home/rafael/evidencia-IR001-2026-07-13.tar.gz` (944K) containing alerts.json + archives.log | 20:15 |
| Hash verified | SHA256: `bca728279c39679df6020f1fa2a4c4be728de5e8c48bbc7b87f249fc3b4aa498` | 20:15 |

**Evidence integrity:** The SHA256 hash above confirms the evidence package has not been modified since collection. Any future copy of this file should produce the same hash when verified with `sha256sum`.

---

## 12. References

- MITRE ATT&CK T1110: https://attack.mitre.org/techniques/T1110/
- MITRE ATT&CK T1046: https://attack.mitre.org/techniques/T1046/
- CIS Benchmark — Windows Server 2019
- Wazuh Rules: 60122, 92652

---

*Report generated by: Rafael — SOC Homelab*  
*Environment: midominio.local | Wazuh 4.x | pfSense 2.7.2*
