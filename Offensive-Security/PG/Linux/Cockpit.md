# NMAP

```
sudo nmap -sC -sV 192.168.236.10  -oN nmap-cockpit
```

![[PG/Linux/attachments/Pasted image 20250710133057.png]]

All ports 

```
sudo nmap -p- -sC -sV 192.168.236.10  -oN nmap-cockpit
```

![[PG/Linux/attachments/Pasted image 20250710142411.png]]
- unfortunately nothing here
# Web (9090)

![[PG/Linux/attachments/Pasted image 20250710133252.png]]
- Login page 
- Blaze server
- admin:admin doesn't work
- Googling defaults for this blaze server: `ubuntu:ubuntu`

Unfortunately these defaults don't work either 

![[PG/Linux/attachments/Pasted image 20250710133519.png]]


## Ferox 

```
feroxbuster -u https://192.168.236.10:9090/ -k -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ferox.9000
```

Let this run and take a look at other site 

![[PG/Linux/attachments/Pasted image 20250710134033.png]]
- this /ping directory is interesting

![[PG/Linux/attachments/Pasted image 20250710134111.png]]

## Search for public exploits for this cockpit service 

![[PG/Linux/attachments/Pasted image 20250710134241.png]]
- this looks promising.
- not sure which version of cockpit this is (I guess 198?)

It looks like a login bypass thing. 
![[PG/Linux/attachments/Pasted image 20250710134431.png]]

Maybe I can enter this payload and see

![[PG/Linux/attachments/Pasted image 20250710134523.png]]

The web app gives an error

![[PG/Linux/attachments/Pasted image 20250710134603.png]]

This other options dropdown also looks interesting 

![[PG/Linux/attachments/Pasted image 20250710134803.png]]
- tried to just add 127.0.0.1 but it prompted for user:pass
- added `ubuntu:ubuntu` but got this error

![[PG/Linux/attachments/Pasted image 20250710134930.png]]

Returning to Ferox output and we have some more directories 

![[PG/Linux/attachments/Pasted image 20250710135047.png]]

Let's check these out before pursuing some authentication bypass with Burp.

Okay, so all of these redirect to the login page.
![[PG/Linux/attachments/Pasted image 20250710135216.png]]

Back to the authentication bypass thing. 

## Burpsuite 

This is what I'll intercept
![[PG/Linux/attachments/Pasted image 20250710135413.png]]
- it was not intercepted + no parameters in Burp.

## Try to hit endpoints with curl

Base
```
curl -k https://192.168.236.10:9090/
```

![[PG/Linux/attachments/Pasted image 20250710142838.png]]

/ping
```
curl -k https://192.168.236.10:9090/ping
```

![[PG/Linux/attachments/Pasted image 20250710142950.png]]

/socket
```
curl -k https://192.168.236.10:9090/socket
```

![[PG/Linux/attachments/Pasted image 20250710143042.png]]

![[PG/Linux/attachments/Pasted image 20250710143325.png]]

Intercepting request....

This stands out 

![[PG/Linux/attachments/Pasted image 20250710143412.png]]

![[PG/Linux/attachments/Pasted image 20250710143432.png]]
- Maybe I can generate a cookie to work? 
  
Okay, well this exploit pops again...  

![[PG/Linux/attachments/Pasted image 20250710143617.png]]
- Let's try this

![[PG/Linux/attachments/Pasted image 20250710143832.png]]
- it didn't work

It looks like it didn't work because this site is https and the exploit code appends http
![[PG/Linux/attachments/Pasted image 20250710144039.png]]

Let's modify the code to be https

![[PG/Linux/attachments/Pasted image 20250710144136.png]]

Re-run it - still fails

![[PG/Linux/attachments/Pasted image 20250710144208.png]]

Maybe try scanning for internal server?
![[PG/Linux/attachments/Pasted image 20250710144309.png]]

![[PG/Linux/attachments/Pasted image 20250710144501.png]]
- still no
# Web (Port 80)

![[PG/Linux/attachments/Pasted image 20250710133746.png]]
- clicked around a bit but no links work

## Ferox 

```
feroxbuster -u http://192.168.236.10/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ferox.80
```

Some immediate results 

![[PG/Linux/attachments/Pasted image 20250710133957.png]]
- I'll want to enum this /js/ directory as well 

Running ferox against the /js

```
feroxbuster -u http://192.168.236.10/js/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ferox.js -scan-dir-listings
```

![[PG/Linux/attachments/Pasted image 20250710144737.png]]
- nope

From the hints-- looks useful to re-enumerate Port 80 with -x php,txt extensions


## Re-running Ferox (Port 80)

```
feroxbuster -u http://192.168.236.10/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt -o ferox.rerun
```

We immediately see the login.php page

![[PG/Linux/attachments/Pasted image 20250710145302.png]]

### Ermahgerd another login!

![[PG/Linux/attachments/Pasted image 20250710145350.png]]
- admin:admin fails
🗒 Note to self--always be re-running scans with likely file extensions!!


Looks like we can finally confirm a hostname

```
 blaze.offsec
```
- add to /etc/hosts

Possible username?
```
JDgodd
```

Tried this payload in both fields and get a helpful error message

```
admin'
```

![[PG/Linux/attachments/Pasted image 20250710145718.png]]

Let's try '%admin'% OR 1=1

![[PG/Linux/attachments/Pasted image 20250710145902.png]]


Oh my lord lol

![[PG/Linux/attachments/Pasted image 20250710145924.png]]
- that's fun

So maybe valid syntax but input is being validated 

```
admin' OR 1=1 
```
- also blocked

Okay so this is a diff response 

```
'''''''''''''UNION SELECT '2
```
![[PG/Linux/attachments/Pasted image 20250710150312.png]]

''''''''''''"SELECT '2

![[PG/Linux/attachments/Pasted image 20250710150758.png]]

# Successful Login

Payload 

```
admin' #
```
![[PG/Linux/attachments/Pasted image 20250710151236.png]]

I basically googled the error response from the application and asked google ai how an attacker might successfully login

This was my google search 

```
logical sql injection iof this is the error Error: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '%'='%' AND password like '%1=1 AND '%'='%'' at line 1 okay so what payload might actually work
```

![[PG/Linux/attachments/Pasted image 20250710151453.png]]
- literally second suggestion worked. thank you googs. 

Alright NOW. 

We have some usernames and passwords 

![[PG/Linux/attachments/Pasted image 20250710151549.png]]
- Passwords look base64 encoded 

Just throw them into cyberchef with from base64 option

![[PG/Linux/attachments/Pasted image 20250710151701.png]]

```
james:canttouchhhthiss@455152
```

```
cameron:thisscanttbetouchedd@455152
```

Hopefully one of these works with ssh. 

![[PG/Linux/attachments/Pasted image 20250710151959.png]]
- james did not 

![[PG/Linux/attachments/Pasted image 20250710152045.png]]
- neither did cameron. 


# Login to Site at 9090

![[PG/Linux/attachments/Pasted image 20250710152153.png]]

We're in 

![[PG/Linux/attachments/Pasted image 20250710152219.png]]

Clicking around. 

Looks like I can add an ssh key. 

![[PG/Linux/attachments/Pasted image 20250710152436.png]]

# Generate ssh key for James

```
ssh-keygen
```


Paste the .pub key here
![[PG/Linux/attachments/Pasted image 20250710155256.png]]

![[PG/Linux/attachments/Pasted image 20250710155322.png]]

Chmod the private key 

```
chmod 600 id_james
```

Attempt to ssh with key 

```
ssh james@192.168.236.10 -i id_james
```
![[PG/Linux/attachments/Pasted image 20250710155454.png]]

And I'm in.

Grab flag
![[PG/Linux/attachments/Pasted image 20250710155628.png]]

# Enum for priv esc
```
sudo -l
```
![[PG/Linux/attachments/Pasted image 20250710155534.png]]
- we can run tar 👀 this is familiar 

This is a very [helpful guide](https://morgan-bin-bash.gitbook.io/linux-privilege-escalation/tar-wildcard-injection-privesc). 

Here is the tmp directory before
![[PG/Linux/attachments/Pasted image 20250710160324.png]]

Creating the exploit files
![[PG/Linux/attachments/Pasted image 20250710160338.png]]

We have the three files
![[PG/Linux/attachments/Pasted image 20250710160354.png]]




Execute the tar command with sudo 

```
sudo /usr/bin/tar -czvf /tmp/backup.tar.gz *
```

![[PG/Linux/attachments/Pasted image 20250710160533.png]]

We're root 

![[PG/Linux/attachments/Pasted image 20250710160725.png]]

![[PG/Linux/attachments/Pasted image 20250710160756.png]]


What's this 
![[PG/Linux/attachments/Pasted image 20250710160900.png]]

![[PG/Linux/attachments/Pasted image 20250710160935.png]]
heheh