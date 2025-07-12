
**Capstone Lab:**

Get access to _CLIENTWK222_ (VM #3) by connecting to the bind shell on port 4444. Use the methods covered in this Module to elevate your privileges to an administrative user. Enter the flag, which is located in **C:\Users\enterpriseadmin\Desktop\flag.txt**

Connect to bindshell

`nc 192.168.123.222 4444`

`whoami`

![[Pasted image 20250305152715.png]]
Check privs (nothing interesting)

![[Pasted image 20250305152803.png]]

User groups:

`whoami /groups`

![[Pasted image 20250305153105.png]]

Existing users and groups 

`net user` and `net localgroup`

![[Pasted image 20250305153240.png]]
- notice enterpriseadmin target

Operating system, version and architecture

`systeminfo`

![[Pasted image 20250305153500.png]]
- Microsoft Windows 11 OS
- Hostname: CLIENTWK222
- This may be a VM running in VMWare 

- Network information
	- `ipconfig /all`
	![[Pasted image 20250305153738.png]]
		- only one ethernet - ipv4 = 192.168.123.222
	- `route print`
	- `netstat -ano`
![[Pasted image 20250305154024.png]]
- not too interested. there is RDP port listening and SMB.


Installed applications

`cd C:\Program Files`

![[Pasted image 20250305154345.png]]
- nothing terrible interesting

Nothing in diana's downloads either. 

WinPEAS time? 

Executed winpeas.exe

Looks like there's a couple of unquoted service paths
![[Pasted image 20250305155239.png]]

Can just pop a reverse shell in the VMWare directory called 

Possible DLL hijacking
![[Pasted image 20250305155505.png]]

Interesting powershell history...
![[Pasted image 20250305155834.png]]
- looks like $SecurePassword and $Username variables have admin privs... 
- can we read those variables?
- maybe see if we can see the `C:\secpol.cfg` or `C:\..\..\local.sdb`?

Okay... no. Cannot write to the Unquoted path directory `C:\Program Files\VMWare\...`

I don't think anything in WinPEAS is going to help. Maybe will try Bloodhound.

Turns out there's a fuckton of notes in diana's documents folder
![[Pasted image 20250305200338.png]]
Can try to login as alex using the `WelcomeToWinter0121` password.

Logged in as Alex 

![[Pasted image 20250305201119.png]]
Repeat enumeration. no interesting privileges.. 

Also cannot change out of his documents folder.

PSReadlines
![[Pasted image 20250305210233.png]]

He seems to have less access than diana.

Looks like alex is part of Remote Desktop Users and Remote Management users so maybe can enumerate SMB shares

![[Pasted image 20250305210526.png]]
#crackmapexec 

Yeah, no interesting or actionable share access.
`crackmapexec smb 192.168.123.222 -u alex -p "WelcomeToWinter0121" --shares`

![[Pasted image 20250305210936.png]]

Alex does show Pwn3d! for winrm 👀
`crackmapexec winrm 192.168.123.222 -u alex -p "WelcomeToWinter0121"`
![[Pasted image 20250305211251.png]]

Try psexec - it doesn't work. 

Get better shell with evil-winrm

`evil-winrm -i 192.168.123.222 -u alex -p "WelcomeToWinter0121"`

![[Pasted image 20250305211940.png]]

Could probably try winpeas 

Reviewing winpeas. 

vmware unquoted path thing. 

OneDrive folder 

![[Pasted image 20250305212912.png]]
Interesting "tasks" folders
![[Pasted image 20250305213107.png]]

Possible password files (Deadend)
`C:\Users\All Users\Microsoft\UEV\InboxTemplates\RoamingCredentialSettings.xml`

Hidden files or directories

```
     C:\Users\Default
     C:\Users\All Users
     C:\Users\All Users\ntuser.pol
     C:\Users\Default User
     C:\Users\Default
     C:\Users\All Users
     C:\Users\alex\AppData\Local\Temp\edge_BITS_412_1336911083\BITBDBD.tmp

```

RDP in as Alex 
`xfreerdp /u:alex /d:clientwk222  /p:WelcomeToWinter0121 /v:192.168.123.222`

Run this:
List of services with binary path
```
Get-CimInstance -ClassName win32_service | Select Name,State,PathName | Where-Object {$_.State -like 'Running'}
```

Tried to see if password was used on any other account using crackmapexec. It is not. 

Googling this specific OS version
![[Pasted image 20250306160011.png]]

Brings up a priv esc cve on exploitdb.
https://www.exploit-db.com/exploits/51203

Likely the vector because diana mentioned something about the backups in her notes. "Move all backup services to not run as system"

This actually seems to complicated. Continuing to research and find this site:

https://jstnk9.github.io/jstnk9/research/DLL-Hijacking-with-DeviceCensus.exe-on-Windows-11/

Related to DLL hijacking. Also confirmed that there is a `DeviceCensus.exe` file 

![[Pasted image 20250306161452.png]]

`Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname`

![[Pasted image 20250306212719.png]]

maybe need to call the dll `sapi_onecore.dll`

Or `TpmCoreProvisioning.dll`

![[Pasted image 20250306212823.png]]


