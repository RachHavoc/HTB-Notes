```
~/Tools/Recon/nmapAutomator.sh -H 192.168.236.168 -t Port
```

![[PG/Windows/attachments/Pasted image 20250711155807.png]]

![[PG/Windows/attachments/Pasted image 20250711155736.png]]

```
~/Tools/Recon/nmapAutomator.sh -H 192.168.236.168 -t All
```

![[PG/Windows/attachments/Pasted image 20250711193411.png]]

![[PG/Windows/attachments/Pasted image 20250711193439.png]]

![[PG/Windows/attachments/Pasted image 20250711193515.png]]

# SMB

![[PG/Windows/attachments/Pasted image 20250711193651.png]]
- domain Fishyyy

![[PG/Windows/attachments/Pasted image 20250711194042.png]]

# Web (4848)

![[PG/Windows/attachments/Pasted image 20250711194230.png]]
- GlassFish login page
- admin:admin didn't work


## Ferox 

```
feroxbuster -u http://192.168.236.168:4848/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ferox.4848
```


# RPC 

```
rpcclient -U '' 192.168.236.168
```

![[PG/Windows/attachments/Pasted image 20250711194641.png]]
- nope

# Web (6060)

![[PG/Windows/attachments/Pasted image 20250711194952.png]]
- SynaMan 5.1 

## Ferox

```
feroxbuster -u http://192.168.236.168:6060/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ferox.6060
```


# Web (8080)

![[PG/Windows/attachments/Pasted image 20250711195614.png]]
- none of the links work
- GlassFish Server Open Source Edition 4.1 
## Ferox 



# Web (8181)

![[PG/Windows/attachments/Pasted image 20250711200144.png]]
- looks the sameeee

## Ferox 

```
feroxbuster -u https://192.168.236.168:8181/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k -o ferox.8181
```


There is a directory traversal vulnerability with GlassFish 4.1. Refer: https://www.exploit-db.com/exploits/39441

https://medium.com/@vivek-kumar/offenisve-security-proving-grounds-walk-through-fish-fccc07ec3b0f

Use this to get admin key file

Target file: glassfish4/glassfish/domains/domain1/config/admin-keyfile

^ Uncrackable admin creds 

# Port (6060)

Synaman has this vuln https://www.exploit-db.com/exploits/45387

SMTP file disclosure.

Use glass fish exploit to read the smtp file creds. Obtain credentials and login via rdp.



# Priv Esc Against a AV

![[PG/Windows/attachments/Pasted image 20250711203528.png]]

Create malicious dll 

![[PG/Windows/attachments/Pasted image 20250711203628.png]]

