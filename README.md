# 🛡️ SOC Home Lab — Wazuh SIEM with Active Directory Attack & Detection

A self-built, fully virtualized Security Operations Center (SOC) lab used to simulate real-world attacks against a Windows Active Directory environment and detect them with a Wazuh SIEM. Built from scratch on a single workstation to practice blue-team / Tier 1 SOC analyst skills: log collection, threat hunting, detection engineering, and incident analysis.

> **Why I built this:** To go beyond certifications (CompTIA A+, Network+, Security+) and get hands-on with the actual day-to-day work of a SOC analyst — collecting telemetry, hunting through alerts, and engineering detections for attacker behavior.

---

## 📐 Architecture

All machines run as KVM/QEMU virtual machines on an isolated NAT network (`labnet`, `10.10.10.0/24`). Nothing in the lab touches the host network or the internet-facing LAN beyond package updates.

```
                         ┌─────────────────────────────┐
                         │      Host: Bazzite-DX        │
                         │   Ryzen 7 3700X / 32GB RAM   │
                         │     KVM / QEMU / libvirt     │
                         └─────────────┬───────────────┘
                                       │  labnet 10.10.10.0/24 (NAT)
         ┌───────────────┬─────────────┼──────────────────┬───────────────┐
         │               │             │                  │               │
   ┌───────────┐   ┌───────────┐  ┌───────────┐     ┌───────────┐
   │   DC01    │   │  WIN11-01 │  │   Wazuh   │     │   Kali    │
   │ Win Srv   │   │  Win 11   │  │  SIEM     │     │  Attacker │
   │  2025     │   │ Enterprise│  │ Ubuntu    │     │           │
   │ AD / DNS  │   │ Endpoint  │  │  24.04    │     │           │
   │.10.10.10.5│   │.10.10.10.20│ │.10.10.10.10│    │.10.10.10.50│
   └───────────┘   └───────────┘  └───────────┘     └───────────┘
        │                │              ▲                  │
        │ Sysmon +       │ Sysmon +     │ logs             │ attacks
        │ Wazuh agent ───┼ Wazuh agent ─┘                  │
        └────────────────┴─────────────────────────────────┘
```

### Host & Hypervisor
| Component | Detail |
|---|---|
| Host OS | Bazzite-DX (Fedora Atomic, immutable) |
| CPU / RAM | AMD Ryzen 7 3700X (8c/16t) / 32 GB |
| Virtualization | KVM / QEMU via `virt-manager` (libvirt) |
| Lab network | `labnet` — `10.10.10.0/24`, NAT, isolated |

### Virtual Machines
| Host | Role | OS | IP | Resources |
|---|---|---|---|---|
| **DC01** | Domain Controller / DNS (`lab.local`) | Windows Server 2025 | `10.10.10.5` | 2 vCPU / 4 GB |
| **WIN11-01** | Domain-joined employee endpoint (victim) | Windows 11 Enterprise | `10.10.10.20` | 2 vCPU / 4 GB |
| **Wazuh** | SIEM — manager, indexer, dashboard | Ubuntu Server 24.04 | `10.10.10.10` | 4 vCPU / 8 GB |
| **Kali** | Attacker | Kali Linux | `10.10.10.50` | 2 vCPU / 4 GB |

---

## 🧰 Tech Stack

- **SIEM:** Wazuh 4.14 (all-in-one: manager + indexer + dashboard)
- **Endpoint telemetry:** Wazuh agent + Sysmon (SwiftOnSecurity config)
- **Directory services:** Active Directory Domain Services (`lab.local`), DNS, Group Policy
- **Attacker tooling:** Nmap, CrackMapExec, smbclient (Kali)
- **Detection:** Windows Security event logs, Sysmon, custom Wazuh rules, MITRE ATT&CK mapping

---

## 🏗️ Build Summary

| Phase | What was built |
|---|---|
| 1 | Enabled CPU virtualization (SVM) and rebased host to Bazzite-DX for built-in KVM/QEMU tooling |
| 2 | Created isolated `labnet` virtual network and dedicated VM storage pool |
| 3 | Deployed Wazuh SIEM (all-in-one) on Ubuntu Server |
| 4 | Built Windows 11 endpoint; installed Sysmon + Wazuh agent; validated telemetry |
| 5 | Stood up Windows Server 2025 as a Domain Controller (`lab.local`); created OU + domain users; joined WIN11-01 to the domain |
| 6 | Deployed Kali attacker VM on the lab network |
| 7 | Ran attack scenarios and performed detection / threat hunting in Wazuh |

Remote dashboard access from a separate analyst laptop was provided via an **SSH tunnel** (`ssh -L 8443:10.10.10.10:443`), since the lab network is isolated.

---

## ⚔️ Attack Scenarios & Detections

Each scenario was executed from the Kali attacker box and then investigated from the Wazuh dashboard — the full offensive → defensive loop.

### 1. Reconnaissance — Network & Service Discovery
**Technique:** `T1046` – Network Service Discovery
```bash
nmap -sn 10.10.10.0/24          # host discovery
nmap -sV -O 10.10.10.5          # service/version + OS fingerprint
nmap -A 10.10.10.5              # aggressive scan
```
Identified the domain controller by its service fingerprint — Kerberos (88), LDAP (389/636), Global Catalog (3268), DNS (53), SMB (445) — and confirmed the domain `lab.local`.

> **Blue-team lesson:** Host-based agents (Wazuh/Sysmon) are weak at detecting port scans, because scan traffic is inbound and filtered as noise. Catching scans reliably requires a **network IDS** (e.g., Suricata/Snort). Documented as a planned improvement.

### 2. Credential Access — Password Brute Force
**Technique:** `T1110` – Brute Force
```bash
crackmapexec smb 10.10.10.5 -u jsmith -p wrongpass_list.txt
```
Generated a burst of failed domain logons.

**Detection:** Windows **Event ID 4625** (failed logon), surfaced in Wazuh with the attacker source IP (`10.10.10.50`), target account (`jsmith`), and failure reason. Repeated failures were correlated into a higher-severity authentication-failure alert.

📸 *Screenshot: `screenshots/04-bruteforce-dashboard.png` — 7 authentication failures clustered in time*

### 3. Initial Access — Valid Credentials
**Technique:** `T1078` – Valid Accounts
```bash
crackmapexec smb 10.10.10.5 -u jsmith -p 'Welcome2026!'   # [+] success
```
**Detection:** Windows **Event ID 4624** (successful logon), logon type 3 (network), from the attacker IP — the pivot from "attempted" to "compromised."

### 4. Collection & Exfiltration — Data from a Network Share
**Techniques:** `T1021.002` – SMB/Windows Admin Shares, `T1039` – Data from Network Shared Drive
```bash
crackmapexec smb 10.10.10.20 -u jsmith -p 'Welcome2026!' --shares
smbclient //10.10.10.20/CompanyData -U 'LAB/jsmith%Welcome2026!' \
  -c 'get Q4_Financials.txt /tmp/loot.txt'
```
A misconfigured, world-readable SMB share (`CompanyData`) was discovered and a "confidential" file exfiltrated using only valid credentials — no exploit required, which mirrors how most real breaches occur.

**Detection pipeline (detection engineering):**
- Enabled Windows **object-access auditing** (`auditpol` — Detailed File Share).
- **Tuned the Wazuh agent** to forward **Event ID 5145**, which the default agent config explicitly filters out as noise.
- Verified the SIEM captures the file-access event with full attribution:

```
EventID 5145 | Subject: jsmith (LAB) | Source: 10.10.10.50
Share: \\*\CompanyData | File: Q4_Financials.txt | Access: ReadData
```
- Authored a custom Wazuh rule mapping this behavior to **MITRE T1039** (see `rules/local_rules.xml`).

📸 *Screenshot: `screenshots/06-5145-event.png` — captured file-access event with attribution*

---

## 🗺️ MITRE ATT&CK Coverage

| Tactic | Technique | ID | Telemetry |
|---|---|---|---|
| Discovery | Network Service Discovery | T1046 | (network IDS — planned) |
| Credential Access | Brute Force | T1110 | Win EventID 4625 |
| Initial Access | Valid Accounts | T1078 | Win EventID 4624 |
| Lateral Movement | SMB / Admin Shares | T1021.002 | Win EventID 5140/5145 |
| Collection | Data from Network Shared Drive | T1039 | Win EventID 5145 (custom rule) |

---

## 🔧 Challenges & Troubleshooting (the real work)

The value of this lab was less in the "happy path" and more in diagnosing and fixing the things that broke. A selection:

- **Virtual TPM failure (`swtpm_setup` exit 1):** Windows 11's required vTPM wouldn't initialize on the immutable host. Traced it through the swtpm log to a missing `/var/lib/swtpm-localca` directory with the wrong ownership; fixed ownership (`tss:tss`) and SELinux context.
- **SELinux / libvirt storage permissions:** VMs on a secondary SSD failed with "permission denied." Resolved by applying the correct SELinux type (`virt_image_t`) to the VM image directory.
- **Ubuntu LVM under-provisioning:** The installer only allocated ~half the virtual disk by default; resized the logical volume so the SIEM had adequate storage.
- **Wazuh agent config corruption:** A config edit saved with the wrong text encoding broke the agent (`Error reading XML file`). Learned to force ASCII encoding on Windows config edits.
- **Group Policy vs. local audit policy:** After domain join, GPO refresh considerations around local `auditpol` settings for object-access auditing.
- **Agent-side event filtering:** The default Wazuh Windows agent **explicitly excludes Event ID 5145** (and other high-volume object-access events) via an event-channel query filter. Identified this as the reason file-access events never reached the manager, and tuned the agent to forward them — a real detection-engineering task.

### 🚧 Known Issue / In Progress
The Event 5145 telemetry is **successfully collected and searchable in the SIEM with full attribution** (user, source IP, filename), confirmed via the Wazuh manager's archive logs and `wazuh-logtest`. A custom alert rule (`rules/local_rules.xml`) was authored to raise a high-severity alert on sensitive-file access, but the rule is not yet rendering as a dashboard alert — currently being debugged (rule chaining / `decoded_as` behavior for Security-channel events). Collection and detection logic are in place; alert presentation is the remaining item.

---

## 📈 Skills Demonstrated

- SIEM deployment and administration (Wazuh)
- Endpoint detection & response telemetry (Sysmon, Wazuh agent)
- Active Directory administration (DC promotion, OUs, users, domain join, GPO)
- Log analysis & threat hunting (Windows Security events, Sysmon)
- Detection engineering (audit policy, agent tuning, custom rules, MITRE mapping)
- Offensive tooling for validation (Nmap, CrackMapExec, SMB)
- Linux & Windows administration, virtualization, networking, SSH tunneling

---

## 🚀 Planned Improvements

- [ ] Add **Suricata** (network IDS) to detect the recon/port-scan phase
- [ ] Resolve custom 5145 alert rendering in the dashboard
- [ ] Add **Atomic Red Team** for broader technique coverage
- [ ] Build detections for Kerberoasting and other AD attacks
- [ ] Integrate **VirusTotal** for malware-detection enrichment
- [ ] Create a dashboard / saved searches for the lab's key detections

---

## 📁 Repository Structure

```
soc-home-lab/
├── README.md
├── rules/
│   └── local_rules.xml        # custom Wazuh detection rule(s)
├── configs/
│   └── sysmonconfig.xml        # Sysmon config reference
└── screenshots/
    ├── 01-vms-running.png
    ├── 02-wazuh-agents.png
    ├── 03-nmap-recon.png
    ├── 04-bruteforce-dashboard.png
    ├── 05-exfil-kali.png
    └── 06-5145-event.png
```

---

*Built and documented by Marcos Fagundes — CompTIA A+ / Network+ / Security+ / ITIL 4 — B.S. Cybersecurity (in progress), WGU.*
