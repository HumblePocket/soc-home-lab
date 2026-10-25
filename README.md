# SOC Home Lab — Wazuh SIEM with Active Directory Attack & Detection

This is a Security Operations Center (SOC) lab I built from scratch to get hands-on with the actual day-to-day work of a security analyst — collecting logs, hunting through alerts, and writing detections for real attacker behavior. I've got my CompTIA A+, Network+, Security+, and ITIL 4, but I wanted something that proves I can actually do the work, not just pass exams. So I built a small company network, attacked it, and then switched sides and tried to catch everything I did.

Everything runs as virtual machines on one workstation, on an isolated network that doesn't touch anything else.

---

## What I Learned

> _[Write this part yourself — in your own words. This is the section hiring managers actually read, and it's the one thing nobody can fake. A few honest sentences are plenty. Some things you could talk about:_
> - _What surprised you about how Wazuh actually collects and processes logs_
> - _The hardest bug you ran into and how you tracked it down_
> - _Why host-based logging alone isn't enough to catch a port scan_
> - _What you'd do differently or add next time_
>
> _Delete this prompt once you've written yours.]_

---

## The Setup

Everything sits on an isolated NAT network, `10.10.10.0/24`. Nothing in the lab can reach the outside world except for pulling updates.

```
                         +-----------------------------+
                         |      Host: Bazzite-DX        |
                         |   Ryzen 7 3700X / 32GB RAM   |
                         |     KVM / QEMU / libvirt     |
                         +--------------+--------------+
                                        |  10.10.10.0/24 (isolated)
        +---------------+---------------+----------------+----------------+
        |               |               |                |
   +-----------+   +-----------+   +-----------+    +-----------+
   |   DC01    |   | WIN11-01  |   |   Wazuh   |    |   Kali    |
   | Win Srv   |   | Win 11    |   |  SIEM     |    | Attacker  |
   |  2025     |   | Endpoint  |   | Ubuntu    |    |           |
   | AD / DNS  |   | (victim)  |   |  24.04    |    |           |
   |.10.10.10.5|   |.10.10.10.20|  |.10.10.10.10|   |.10.10.10.50|
   +-----------+   +-----------+   +-----------+    +-----------+
```

| Host | What it is | OS | IP |
|---|---|---|---|
| DC01 | Domain controller / DNS for `lab.local` | Windows Server 2025 | 10.10.10.5 |
| WIN11-01 | A domain-joined "employee" PC — the victim | Windows 11 Enterprise | 10.10.10.20 |
| Wazuh | The SIEM (manager + indexer + dashboard) | Ubuntu Server 24.04 | 10.10.10.10 |
| Kali | The attacker | Kali Linux | 10.10.10.50 |

The host is a Ryzen 7 3700X with 32GB of RAM running Bazzite-DX, with all the VMs under KVM/QEMU.

![All lab VMs running](screenshots/01-vms-running.png)
*All four VMs running at once — Wazuh, DC01 (Server Manager showing the AD and DNS roles), Windows 11, and Kali.*

![Wazuh agents active](screenshots/02-wazuh-agents.png)
*Both Windows machines reporting into the Wazuh manager.*

---

## What's Running On It

- **SIEM:** Wazuh 4.14 (all-in-one install)
- **Endpoint logging:** Wazuh agent + Sysmon (SwiftOnSecurity config) on both Windows hosts
- **Directory:** Active Directory for `lab.local`, with DNS and a handful of domain users
- **Attacker tools:** Nmap, CrackMapExec, smbclient
- **Detection:** Windows Security logs, Sysmon, and a custom Wazuh rule, mapped to MITRE ATT&CK

I set up the domain with a few normal users plus a privileged `itadmin` account — realistic targets for credential attacks — and joined the Windows 11 box to the domain like a real work laptop.

![Domain users](screenshots/07-ad-users.png)
*The user accounts I created in the `lab.local` domain.*

![Windows 11 domain-joined](screenshots/08-win11-domain-joined.png)
*The Windows 11 endpoint logged in as a domain user (`lab\jsmith`).*

---

## The Attacks I Ran (and How I Caught Them)

For each one I ran the attack from Kali, then switched over to the Wazuh dashboard and tried to find it — the full attacker-to-analyst loop.

### 1. Recon — mapping the network (MITRE T1046)

```bash
nmap -sn 10.10.10.0/24     # what's alive
nmap -A 10.10.10.5         # dig into the DC
```

The scan told me right away that `10.10.10.5` was a domain controller — Kerberos (88), LDAP (389/636), the Global Catalog (3268), DNS (53), and SMB (445) were all open, and it leaked the domain name `lab.local`.

![Nmap recon](screenshots/03-nmap-recon.png)
*The aggressive scan fingerprinting DC01 as a domain controller.*

One thing I learned here: my endpoint agents barely saw this scan. Host-based logging isn't great at catching port scans because the traffic is just inbound noise. To reliably catch recon you need a network IDS like Suricata — which is on my to-do list below.

### 2. Brute force — guessing passwords (MITRE T1110)

```bash
crackmapexec smb 10.10.10.5 -u jsmith -p wrongpass_list.txt
```

This threw a bunch of bad passwords at the domain, which Windows logged as **Event 4625 (failed logon)**. Wazuh picked them up with the attacker's IP, the account being targeted, and the reason — and grouped the repeated failures into a higher-severity alert.

![Brute-force dashboard](screenshots/04-bruteforce-dashboard.png)
*The failed logins stacking up in the Wazuh dashboard, tagged with MITRE techniques.*

![Failed logon detail](screenshots/10-brute-force-event-detail.png)
*One failed-logon event — source IP 10.10.10.50, target jsmith, logon type 3, bad-password substatus 0xC000006A. This is the kind of detail you'd put in a ticket.*

### 3. Getting in with a real password (MITRE T1078)

```bash
crackmapexec smb 10.10.10.5 -u jsmith -p 'Welcome2026!'   # [+] it worked
```

Once a password hit, that showed up as **Event 4624 (successful logon)**, type 3, from the attacker's IP — the moment things go from "someone's trying" to "someone's in."

![CrackMapExec success](screenshots/11-crackmapexec.png)
*The valid login confirmed, plus a wide-open SMB share (CompanyData, READ/WRITE) that shouldn't have been there.*

### 4. Stealing a file (MITRE T1021.002, T1039)

```bash
smbclient //10.10.10.20/CompanyData -U 'LAB/jsmith%Welcome2026!' \
  -c 'get Q4_Financials.txt /tmp/loot.txt'
```

I pulled a fake "confidential" file off that open share using nothing but a valid username and password — no exploit needed, which is how a lot of real breaches actually happen.

![Exfiltration](screenshots/05-exfil-kali.png)
*Pulling the file off the share, with its contents.*

Catching this one took real work. By default Windows doesn't even log file access, and on top of that the Wazuh agent filters out the file-access event (5145) as noise. So I had to:

- Turn on object-access auditing in Windows (`auditpol`)
- Edit the Wazuh agent config to stop filtering Event 5145
- Confirm the SIEM was actually receiving it, with the user, source IP, and filename all attached

![Captured 5145 event](screenshots/06-5145-event.png)
*The file-access event in the SIEM: jsmith, from 10.10.10.50, reading Q4_Financials.txt.*

Then I wrote a custom rule to flag sensitive-file access and map it to MITRE T1039:

![Custom detection rule](screenshots/09-custom-rule.png)
*My custom detection rule in local_rules.xml.*

---

## MITRE ATT&CK Coverage

| Tactic | Technique | ID | What logged it |
|---|---|---|---|
| Discovery | Network Service Discovery | T1046 | (network IDS — planned) |
| Credential Access | Brute Force | T1110 | Event 4625 |
| Initial Access | Valid Accounts | T1078 | Event 4624 |
| Lateral Movement | SMB / Admin Shares | T1021.002 | Event 5140/5145 |
| Collection | Data from Network Shared Drive | T1039 | Event 5145 (custom rule) |

---

## What Broke (and How I Fixed It)

Honestly, this was the part I learned the most from. A few of the things I had to work through:

- **Windows 11 wouldn't create its virtual TPM.** Windows 11 requires a TPM and the VM kept failing to make one. I dug through the swtpm log and found a missing folder with the wrong ownership, fixed the ownership and the SELinux label, and it came up.
- **VMs on my second drive got "permission denied."** Turned out to be SELinux — I had to set the right security label (`virt_image_t`) on the folder holding the VM disks.
- **My Ubuntu VM only used half its disk.** The installer under-allocated the storage, so I resized the logical volume to give the SIEM enough room.
- **I broke the Wazuh agent by saving a config in the wrong encoding.** It wouldn't start and threw an XML error. Lesson learned — force ASCII when editing those configs on Windows.
- **File-access events never showed up in Wazuh.** This one took a while. The event was being logged on Windows but never reached the manager, because the default Wazuh agent config explicitly filters out Event 5145. Once I found and removed that filter, it came through.

### Still working on this

The file-access telemetry (5145) is collected and searchable in the SIEM with the full details — user, source IP, filename — and I confirmed it with the manager's archive logs and `wazuh-logtest`. I wrote a custom rule to raise a high-severity alert on it; the collection and the rule logic are in place, but getting the alert to render cleanly on the dashboard (how custom rules chain onto Security-channel events) is the piece I'm still sorting out.

---

## Still To Do

- [ ] Add Suricata (network IDS) to catch the recon/scan phase
- [ ] Finish the custom 5145 alert so it shows on the dashboard
- [ ] Add Atomic Red Team to cover more techniques
- [ ] Build detections for AD attacks like Kerberoasting
- [ ] Hook in VirusTotal for malware detection

---

## Repo Layout

```
soc-home-lab/
  README.md
  rules/
    local_rules.xml        # my custom Wazuh detection rule
  configs/
    sysmonconfig.xml        # Sysmon config
  screenshots/              # evidence from the lab
```

---

*Marcos Fagundes — CompTIA A+ / Network+ / Security+ / ITIL 4 — B.S. Cybersecurity (in progress), WGU.*
