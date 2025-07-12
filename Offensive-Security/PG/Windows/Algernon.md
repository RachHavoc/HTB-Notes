# NMAP

```
sudo nmap -sC -sV 192.168.x.x -oN nmap-algernon
```

![[PG/Windows/attachments/Pasted image 20250702195035.png]]

All ports has more open (including 17001)
![[PG/Windows/attachments/Pasted image 20250702205612.png]]
# FTP

![[PG/Windows/attachments/Pasted image 20250702195147.png]]

Recursively download these directories 
```
wget -r -l 0 ftp://anonymous:anonymous@192.168.184.65/Logs/*
```

![[PG/Windows/attachments/Pasted image 20250702200222.png]]
- no paswords
- there is a user called admin
# Port 80 
IIS Default 


# Port 9998 
Login 

![[PG/Windows/attachments/Pasted image 20250702200823.png]]

# Search for exploits for web app

Searching for exploits for SmarterMail

Found this RCE exploit https://www.exploit-db.com/exploits/49216

Yeah. This exploit worked. 

![[PG/Windows/attachments/Pasted image 20250702210334.png]]

Woohoo! Got initial access

I am root 

![[PG/Windows/attachments/Pasted image 20250702210416.png]]

Root flag: 7ca8e8fce7b4bab071f4405ca8b356f4