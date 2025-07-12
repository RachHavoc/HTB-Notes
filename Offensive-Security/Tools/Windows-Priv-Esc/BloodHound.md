Exists within the #SharpCollection 

GitHub:
https://github.com/Flangvik/SharpCollection

CMD + F for "SharpHound" to see what file it is in 
![[Pasted image 20241223225127.png]]
![[Pasted image 20241223225108.png]]

It's in here 

![[Pasted image 20241223225241.png]]

Find SharpHound.exe and download 

![[Pasted image 20241223225355.png]]

https://github.com/Flangvik/SharpCollection/tree/master/NetFramework_4.7_Any

Move SharpHound.exe to remote machine and execute it with 

`.\SharpHound.exe`

If Windows Defender blocks it.. 

Download #bloodhoundingestor 

https://github.com/dirkjanm/BloodHound.py

#Bloodhound Releases also available here:

https://github.com/SpecterOps/BloodHound-Legacy/releases

Download the release compatible with attacker machine OS (probably linux x64)

Unzip the file. 

Execute with `./Bloodhound-linux-x64/Bloodhound`

Change bloodhound / neo4j password 

```
/etc/bhapi/bhapi.json
```

(penny:penny)

