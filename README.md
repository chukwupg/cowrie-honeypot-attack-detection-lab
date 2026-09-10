# Network Attack Detection & Reporting (Honeypot Lab)

## Project Overview
This project demonstrates a simulated cyber attack on a honeypot environment and the process of detecting, capturing, and analyzing malicious activity.

The lab was built using two virtual machines:
- **Kali Linux (Attacker)**
- **Ubuntu Server with Cowrie Honeypot (Target)**

This README is written to be **reusable**. Follow it top to bottom on a fresh set of VMs and you'll reproduce the same lab, including fixes for a couple of challenges hit during the original build.

---

## Objective

Simulate an attacker scanning and brute-forcing an exposed SSH service, and demonstrate that the honeypot correctly detects, logs, and provides evidence of the attack (source IP, credentials tried, commands run, session recordings).

### Key Activities
- Deploy a functional honeypot using Cowrie
- Simulate attacks using Nmap and SSH brute-force
- Capture network traffic using Wireshark
- Analyze logs and generate an incident report

---

## Architecture

```
┌──────────────────────┐        Isolated host-only network         ┌────────────────────────────┐
│   Kali Linux VM      │  ───────────────────────────────────▶         Ubuntu Server VM        
│   (Attacker)         │        192.168.72.0/24                    │   (Honeypot target)        │
│   192.168.72.11      │                                           │   192.168.72.10            │
│                      │                                           │                            │
│  - Nmap              │                                           │  - Cowrie (port 2222)      │
│  - Wireshark         │                                           │  - Real OpenSSH (port 22)  │
│  - Hydra             │                                           │                            │
└──────────────────────┘                                           └────────────────────────────┘
```

The two VMs sit on an **isolated VMware host-only network** with no internet access, making it safe to attack without risking anything outside the lab.

---

## Prerequisites

- VMware Workstation 
- Ubuntu Server LTS ISO (minimal install, no desktop)
- Kali Linux prebuilt VMware image (from kali.org)
- Host machine with at least 8GB RAM (Kali: 2GB or 4GB / Honeypot: 2GB, leaves headroom for the host)

---

## Step 1: Create an isolated lab network

1. In VMware: **Edit - Virtual Network Editor - Add Network - VMnet2** (Or any desired Network) 
2. Set type to **Host-only** to create an isolated lab network (Optional: **rename: honeypot-isolated** for easy identification).
3. Choose a distinct subnet (`192.168.72.0/24`).
4. Assign **both VMs** to this network after using NAT network to install the necessary tools and dependencies below. 

> ⚠️ Avoid VMware's default NAT network (`VMnet8`) if you want a truly isolated segment, NAT still routes to the internet, which isn't necessary for this lab and adds noise your report doesn't need.

### Evidence

**Host-Only Network Created**

![Isolated Network](/screenshots/create-isolated-network.png)

---

## Step 2: Build the honeypot VM (Ubuntu Server + Cowrie)

**2a. Create the VM**
- Ubuntu Server LTS (minimal), 2GB RAM, 2 CPUs, 20GB disk.
- Network adapter: use NAT at first to install the dependencies and tools, then power off the machine and assign it to the isolated host-only network created above.

**2b. Install Cowrie (Using pip method to avoid the `bin/cowrie` path issue)**

```bash
sudo apt update && sudo apt install -y git python3-venv python3-dev python3-pip libssl-dev libffi-dev build-essential authbind
sudo adduser --disabled-password cowrie
sudo su - cowrie
python3 -m venv cowrie-env
source cowrie-env/bin/activate
pip install cowrie
```

> **Challenge:** if you would rather `git clone` the Cowrie repo at http://github.com/cowrie/cowrie, the launcher lives at `bin/cowrie` relative to the repo root. With `pip install cowrie`, the `cowrie` command is installed directly into your active virtualenv's `bin/` and is **not** prefixed with `bin/` so running `bin/cowrie start` after a plain pip install will fail with "file or directory not found." Just run `cowrie start` (with the venv active).

**2c. Start and verify**

```bash
cowrie start
cowrie status
tail -f var/log/cowrie/cowrie.log
```

You should see `CowrieSSHFactory starting on 2222` and `Ready to accept SSH connections`. 

> **Challenge:** the venv activation does **not** persist across a VM reboot. If you power off/on the VM, you must re-run `sudo su - cowrie` and `source cowrie-env/bin/activate` before `cowrie start` will be found on PATH. `cowrie status` alone won't tell you it's a PATH issue, it'll just say "command not found."

**2d. Note the honeypot's IP**

```bash
ip a
```

**2e. (Note this If you'll `scp` logs off this VM later)** if you created the `cowrie` user with `adduser --disabled-password`, password-based `scp` will fail with "Permission denied" because there's no password set. 

### Fix:

```bash
# Set Password for user `cowrie` 
sudo passwd cowrie
```

### Evidence

**Ubuntu Server VM Created and Running**

![Honeypot VM Running](/screenshots/honeypot-isolated-network-confirmation.png)

**Cowrie Honeypot Installed**

![Honeypot Installed](/screenshots/cowrie-installed.png)

**Cowrie Honeypot Running**

![Cowrie](/screenshots/honeypot-running-on-isolated-network.png)

---

## Step 3: Build the Kali attacker VM

1. Import the prebuilt Kali `.vmx` into VMware.
2. Set its network adapter to the **same isolated host-only network** as the honeypot.
3. Allocate 2GB or 4GB RAM, 2 CPUs.
4. Boot, and confirm connectivity:

```bash
ip a
ping -c 4 <honeypot-ip>
```

### Evidence

**Kali - Honeypot Connectivity Test**

![Conectivity Test](/screenshots/kali-and-honeypot-recheability-test.png)
---

## Step 4: Capture traffic and run the attack

**Start Wireshark first**, on the interface facing the honeypot network, *before* initiating attack.

```bash
sudo wireshark
```

**Nmap reconnaissance:**

```bash
mkdir -p ~/honeypot-project/nmap-results && cd ~/honeypot-project/nmap-results

nmap -sT <honeypot-ip> -oN basic_scan.txt          # TCP connect scan
nmap -sV <honeypot-ip> -oN service_scan.txt        # Service/version detection
sudo nmap -O <honeypot-ip> -oN os_scan.txt         # OS fingerprint
sudo nmap -sC -sV <honeypot-ip> -oN script_scan.txt
sudo nmap -A -p- <honeypot-ip> -oA full_scan       # Full combined scan (.nmap/.xml/.gnmap)
```

**Manual SSH login attempts (Cowrie logs these regardless of the password):**

```bash
ssh root@<honeypot-ip> -p 2222
ssh admin@<honeypot-ip> -p 2222
ssh test@<honeypot-ip> -p 2222
```

**Automated brute-force with Hydra:**

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt -t 4 ssh://<honeypot-ip>:2222 -f
```

**Stop Wireshark**, and export capture file, **File - Save As**  (`honeypot_capture.pcap`)

### Evidence

**Nmap Recon Result**
![Nmap Recon Redacted](/screenshots/redacted-nmap-result.png)

**Manual SSH Login Attempt**
![Manual SSH Login Attempt](/screenshots/manual-ssh-login-attempt.png)

**Automated Brute-force with Hydra**
![Brute-force attemt](/screenshots/bruteforce-attempt-hydra.png)

**Wireshark**
![Wireshark](/screenshots/wireshark-port-2222-traffic.png)

---

## Step 5: Log Assembling

From the honeypot VM:

```bash
cat ~/cowrie/var/log/cowrie/cowrie.log     # human-readable 
cat ~/cowrie/var/log/cowrie/cowrie.json    # structured, one JSON event per line
```

Useful filters:

```bash
grep -E "login.(success|failed)" cowrie.json

# If you have jq installed:
cat cowrie.json | jq 'select(.eventid | test("login")) | {timestamp, src_ip, username, password, eventid}'
```

>**I copied these logs off to my host machine for assembling, log analysis, and reporting.**

```bash
scp cowrie@<honeypot-ip>:~/cowrie/var/log/cowrie/cowrie.json C:\Users\USER\Documents
scp cowrie@<honeypot-ip>:~/cowrie/var/log/cowrie/cowrie.log C:\Users\USER\Documents
```

### Evidence

**cowrie.log**
![cowrie.log](/screenshots/cowrie.log-output.png)

**cowrie.json**
![cowrie.json](/screenshots/cowrie.json-output.png)

**Filtered log output (grep)**
![grep filter](/screenshots/filtered-log.png)

**Exporting Deliverables using SCP**
![Deliverables](/screenshots/export-deliverables.png)

---

## Incident Summary
 
### **What happened:** 
A Kali Linux VM (`192.168.72.11`) scanned and attacked a Cowrie SSH honeypot (`192.168.72.10:2222`) on an isolated lab network. Nmap reconnaissance was followed by manual login attempts and an automated Hydra brute-force. The honeypot accepted every credential submitted (by design) and logged full session detail; the real OpenSSH service on port 22 correctly rejected the same root login attempt, confirming it was properly hardened.
 
- **Attacker IP:** 192.168.72.11
- **Target IP:** 192.168.72.10 (Cowrie honeypot, port 2222)
- **Credentials captured:** `root/admin123`, `root/toor`, `admin/12345`, `test/12345`, `root/123456789`, `root/12345`, `root/password`
- **Automated-tool signature:** 3 Hydra logins accepted within the same second, distinct from the manual attempts seconds apart
- **Evidence collected:** Nmap scan output, full Wireshark packet capture, Cowrie JSON/text logs (including TTY session recordings), honeypot-running screenshot
- **Recommendation:** enforce key-based SSH auth, deploy fail2ban for rate-limiting/lock out repeated login failures, and monitor for burst-login patterns as an automated-attack indicator

📄 **Full report:** [`report/Incident_Report_White.pdf`](report/Incident_Report.pdf): 
A 1-page report with the complete attack timeline, indicators of compromise, and defense recommendations.


**Navigate to `report/Incident_Report.pdf` for full incident report.**

---

## Repository structure

```
honeypot-attack-detection/
├── README.md
├── screenshots/
│   
├── nmap-results/
│   ├── basic_scan.txt
│   ├── service_scan.txt
│   ├── os_scan.txt
│   ├── script_scan.txt
│   └── full_scan.nmap / .xml / .gnmap
├── pcap/
│   └── honeypot_capture.pcap
├── logs/
│   ├── cowrie.log
│   └── cowrie.json
└── report/
    └── Incident_Report.pdf
```

---

## Key results from this lab

| Item | Value |
|---|---|
| Attacker IP | 192.168.72.11 |
| Honeypot IP | 192.168.72.10 |
| Real SSH (port 22) | Correctly rejected root login |
| Honeypot SSH (port 2222) | Accepted every credential tried |
| Credentials captured | root/admin123, root/toor, admin/12345, test/12345, root/123456789, root/12345, root/password |
| Automated-tool signature | 3 Hydra logins accepted within the same second |

---

## Disclaimer

This lab was performed entirely within an isolated, non-internet-facing virtual network against systems the author owns and controls. Do not scan, brute-force, or otherwise attack systems you do not own or have explicit authorization to test.

---

