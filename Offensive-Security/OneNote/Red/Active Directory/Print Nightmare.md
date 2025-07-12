==cube0x0 RCE -==  ==https://github.com/cube0x0/CVE-2021-1675==  
==calebstewart LPE -==  ==https://github.com/calebstewart/CVE-2021-1675==
 > From <[https://academy.tcm-sec.com/courses/1152300/lectures/33637715](https://academy.tcm-sec.com/courses/1152300/lectures/33637715)>   
Testing for this vulnerable and the results indicate that it is.

![Exported image](Exported%20image%2020241208212358-0.png)  

Mitigation:  
Disable Spooler service
 
Attacc:
 
Install latest version of impacket
 
Get python script.
 
Create msfvenom payload
 ![Exported image](Exported%20image%2020241208212358-1.png)  

Must do msfconsole as well.
 
use multi/handler (metasploit listener)
 ![Exported image](Exported%20image%2020241208212359-2.png)  

set payload to the msfvenom script just created. ￼

![Exported image](Exported%20image%2020241208212359-3.png)  

Set lhost to 10.0.0.95 & lport to 5555(same as msfvenom)
 ![Exported image](Exported%20image%2020241208212403-4.png)  

Set up file share to share the shell.dll created with msfvenom:
 
```
smbserver.py share 'pwd'











```
 ![Exported image](Exported%20image%2020241208212404-5.png)  
![Exported image](Exported%20image%2020241208212404-6.png)  

Now good to go.
 ![Exported image](Exported%20image%2020241208212405-7.png)  
 Domain Controller IP

If error code add SMB2support to the smbserver share
 ![Exported image](Exported%20image%2020241208212406-8.png)  

Error?
 ![Exported image](Exported%20image%2020241208212407-9.png)  

Nope….
 
Impacket can't find payload