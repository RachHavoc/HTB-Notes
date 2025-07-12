![Exported image](Exported%20image%2020241208212527-0.octet-stream)

Question: My enum4linux and/or smbclient are not working. I am receiving "Protocol negotiation failed: NT_STATUS_IO_TIMEOUT". How do I resolve?
 
Resolution:  
On Kali, edit /etc/samba/smb.conf  
Add the following under global:  
client min protocol = CORE  
client max protocol = SMB3

**SMB port 139** is a file sharing protocol  
Commonly used in work environments
 
Searching for exploits within metasploit:

- Metasploit Auxiliary modules are used for enumeration
- Looking for SMB version information
    
    - Module: auxiliary/scanner/smb/smb_version

![Exported image](Exported%20image%2020241208212527-1.png) ![Exported image](Exported%20image%2020241208212528-2.png)

Info Gathered from Kioptrix machine

Identified Samba 2.2.1a for Kioptrix machine

**Utilize smbclient to attempt to connect to victim machine:**  
smbclient -L [\\\\Target IP\\](file:///\\\\Target IP\\)
 ![Exported image](Exported%20image%2020241208212529-3.png)

```
(hit enter at root password prompt if unknown)

Notice two file shares: IPC & Admin

Utilize smbclient to attempt to connect to admin  
smbclient [\\\\Target IP\\ADMIN$](file:///\\\\Target IP\\ADMIN$)

If anonymous access no workie for admin share: attempt to connect to IPC share:  
smbclient [\\\\Target IP\\IPC$](file:///\\\\Target IP\\IPC$)

```
 
**Notice two file shares: IPC & Admin**
 
**Utilize smbclient to attempt to connect to admin**  
smbclient [\\\\Target IP\\ADMIN$](file:///\\\\Target IP\\ADMIN$)
 
**If anonymous access no workie for admin share: attempt to connect to IPC share:**  
smbclient [\\\\Target IP\\IPC$](file:///\\\\Target IP\\IPC$)