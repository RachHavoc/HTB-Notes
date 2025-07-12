**What is SMB Relay?**  
Instead of cracking hashes gathered with Responder, we can relay those hashes to specific machines and potentially gain access.
 
**Requirements**:

- SMB signing must be disabled on the target
- Relayed user credentials must be admin on machine   
![Exported image](Exported%20image%2020241208212309-0.png)  
    
Results of this scan:

![Exported image](Exported%20image%2020241208212310-1.png)  
![Exported image](Exported%20image%2020241208212310-2.png)  
![Exported image](Exported%20image%2020241208212314-3.png)  

Rerun responder:

- listening for events

Run ntlmrelayx.py with -t <target IP> and smb2support

![Exported image](Exported%20image%2020241208212315-4.png)  

On the victim machine input attacker machine IP.

![Exported image](Exported%20image%2020241208212316-5.png)

Success! Why?? Because SMB signing is disabled.

![Exported image](Exported%20image%2020241208212317-6.png)   
SMB Relay Attack Part 2:
 ![Exported image](Exported%20image%2020241208212319-7.png)  
![Exported image](Exported%20image%2020241208212320-8.png)   
Set up netcat listener and enter into SMB message block.

![Exported image](Exported%20image%2020241208212321-9.png)  

Explore: shares, use ADMIN$, ls

![Exported image](Exported%20image%2020241208212325-10.png)

ntlmrelayx.py -t <target IP> -smb2support -e test.exe  
ntlmrelayx.py -t <target IP> -smb2support -c "whoami"
         
![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-eb581851244045a0910d67b8b9ec36dd!1-5CC9E121EFE9768B!6236/$value)  

**Message signing enabled but not required indicates vulnerability to SMB relay attacks.**
 
Save vulnerable machine Ips to targets.txt file

Responder.conf

- Turn off SMB and HTTP
    
![Exported image](Exported%20image%2020241208212325-12.png)
   

No shell created.
 ![Exported image](Exported%20image%2020241208212326-13.png)