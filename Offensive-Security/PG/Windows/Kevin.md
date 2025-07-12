
# NMAP 
```
sudo nmap -sC -sV 192.168.247.45 -oN nmap-kevin
```

```
kevin / f8uHwN88Sx
```

![[PG/Windows/attachments/Pasted image 20250706005412.png]]

# Web 

Logged in already as admin:admin

![[PG/Windows/attachments/Pasted image 20250706005557.png]]

![[PG/Windows/attachments/Pasted image 20250706005700.png]]
- HP Power Manager 4.2 (Build 7)

# Exploit for initial access

This exploit worked after googling the HP Power Manager 

https://www.rapid7.com/db/modules/exploit/windows/http/hp_power_manager_filename/

![[PG/Windows/attachments/Pasted image 20250706010202.png]]

I'm system already too.

![[PG/Windows/attachments/Pasted image 20250706010343.png]]

