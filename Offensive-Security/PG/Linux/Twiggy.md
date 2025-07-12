# NMAP

```
sudo nmap -sC -sV 192.168.184.62 -oN nmap-twiggy
```

![[PG/Linux/attachments/Pasted image 20250701211912.png]]

Website 
![[PG/Linux/attachments/Pasted image 20250701212501.png]]
- admin:admin no work 


Gobuster doesn't work on port 80 
![[PG/Linux/attachments/Pasted image 20250701212544.png]]

It is working on port 8000 site which is an api 
![[PG/Linux/attachments/Pasted image 20250701212637.png]]
![[PG/Linux/attachments/Pasted image 20250701212616.png]]

Got this file path from burp request and viewing page source
![[PG/Linux/attachments/Pasted image 20250701215928.png]]

![[PG/Linux/attachments/Pasted image 20250701220317.png]]