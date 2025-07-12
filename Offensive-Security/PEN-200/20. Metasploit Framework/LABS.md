3. **Capstone Exercise**: Use the methods and techniques from this Module to enumerate VM Group 1. Get access to both machines and find the flag. Once the VM Group is deployed, please wait two more minutes for one of the web applications to be fully initialized.

VM #1: **192.168.208.225**
VM #2: **192.168.208.226**

Begin scan 15:59 EST - Ended at 16:01
`db_nmap -A 192.168.208.225 192.168.208.226`

`192.168.208.225`
![[Pasted image 20250207160353.png]]
```hlt:5
[*] Nmap: PORT     STATE SERVICE       VERSION
[*] Nmap: 135/tcp  open  msrpc         Microsoft Windows RPC
[*] Nmap: 139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
[*] Nmap: 445/tcp  open  microsoft-ds?
[*] Nmap: 8080/tcp open  http          Jetty 9.4.48.v20220622
[*] Nmap: |_http-open-proxy: Proxy might be redirecting requests
[*] Nmap: |_http-title: NiFi
[*] Nmap: |_http-server-header: Jetty(9.4.48.v20220622)
[*] Nmap: No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
[*] Nmap: TCP/IP fingerprint:
```

Want to check for exploits for this web app:
Jetty 9.4.48.v20220622

`192.168.208.226`

![[Pasted image 20250207160607.png]]

```hlt:7
*] Nmap: Nmap scan report for 192.168.208.226
[*] Nmap: Host is up (0.074s latency).
[*] Nmap: Not shown: 997 closed tcp ports (reset)
[*] Nmap: PORT    STATE SERVICE       VERSION
[*] Nmap: 135/tcp open  msrpc         Microsoft Windows RPC
[*] Nmap: 139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
[*] Nmap: 445/tcp open  microsoft-ds?
[*] Nmap: No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
```


Discovered services

```
msf6 > services
```

![[Pasted image 20250207160719.png]]
Auxiliary scanner for SMB (for fun)

```
msf6 auxiliary(scanner/smb/smb_version) > services -p 445 --rhosts
```

```
msf6 auxiliary(scanner/smb/smb_version) > run
```

Results: 
192.168.208.225 - using SMB version 2,3
192.168.208.226 - using SMB version 2,3 
```
msf6 auxiliary(scanner/smb/smb_version) > run
[*] 192.168.208.225:445   - SMB Detected (versions:2, 3) (preferred dialect:SMB 3.1.1) (compion capabilities:LZNT1, Pattern_V1) (encryption capabilities:AES-256-GCM) (signatures:option(guid:{b7e47211-bbc6-473f-b2e8-7b6ea8671530}) (authentication domain:ITWK03)
[+] 192.168.208.225:445   -   Host is running Version 10.0.22000 (likely Windows 11 version )
[*] Scanned 1 of 2 hosts (50% complete)
[*] Scanned 1 of 2 hosts (50% complete)
[*] 192.168.208.226:445   - SMB Detected (versions:2, 3) (preferred dialect:SMB 3.1.1) (compion capabilities:LZNT1, Pattern_V1) (encryption capabilities:AES-256-GCM) (signatures:option(guid:{4cfcb6d3-9883-4f78-b4f0-a3032a799cd9}) (authentication domain:ITWK04)
[+] 192.168.208.226:445   -   Host is running Version 10.0.22000 (likely Windows 11 version )
[*] Scanned 2 of 2 hosts (100% complete)
[*] Auxiliary module execution completed
```

`search type:auxiliary jetty`

old exploit didn't work

looking at webpage:

![[Pasted image 20250207163024.png]]

Apache NiFi

Testing out HTTP Directory Brute Force Scanner

`use auxiliary/scanner/http/brute_dirs`

This didn't work.. there were no actions?

```
gobuster dir -u 'http://192.168.208.225:8080/nifi' -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -o dirbust-out
```

Some interesting directories

![[Pasted image 20250207164759.png]]

login page won't show up without SSL

well this is useful - poking at site 

```
System Diagnostics

NiFi

NiFi Version
    1.18.0-SNAPSHOT
Tag
    nifi-1.15.0-RC3
Build Date/Time
    07/27/2022 22:24:06 PDT
Branch
    main
Revision
    34084d0

Java

Version
    1.8.0_342
Vendor
    Amazon.com Inc.

Operating System

Name
    Windows 11
Version
    10.0
Architecture
    amd64


```

Poking at SMB.

Very verbose error code using #smbclient 

```
┌──(cadet㉿kali)-[~/PEN-200/metasploit]
└─$ smbclient -L 192.168.208.225 -U '' -P
Failed to open /var/lib/samba/private/secrets.tdb
_samba_cmd_set_machine_account_s3: failed to open secrets.tdb to obtain our trust credentials for WORKGROUP
Failed to set machine account: NT_STATUS_INTERNAL_ERROR
```

actually not sure if that error is produced by my box or not. 


