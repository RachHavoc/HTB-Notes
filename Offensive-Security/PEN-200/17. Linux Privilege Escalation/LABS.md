2. **Capstone Lab**: Connect to VM 2 with the provided credentials and gain a root shell by abusing a different kernel vulnerability.

creds: `joe:offsec`
ip: `192.168.131.216`

`cat /etc/issue`
`uname -r`
`arch`
![[Pasted image 20250202135644.png]]
`└─$ searchsploit "linux kernel Ubuntu 16 Local Privilege Escalation"   | grep  "4.4.*"searchsploit "linux kernel Ubuntu 16 Local Privilege`

Ended up running linpeas after several failed c exploits. 

![[Pasted image 20250202151712.png]]

this is interesting

![[Pasted image 20250202151939.png]]

I broke the box with linpeas. 

3. **Capstone Lab**: Connect to the VM 3 with the provided credentials and use an appropriate privilege escalation technique to gain a root shell and read the flag.


4. **Capstone Lab**: On the Module Exercise VM 4, use another appropriate privilege escalation technique to gain access to root and read the flag. Take a closer look at file permissions.

5. **Capstone Lab**: Again, use an appropriate privilege escalation technique to gain access to root and read the flag on the Module Exercise VM 5. Binary flags and custom shell are what to look for.