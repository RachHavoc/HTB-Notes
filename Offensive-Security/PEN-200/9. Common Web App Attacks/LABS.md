**Capstone Lab**: Start the _Future Factor Authentication_ application on VM #3. Identify the vulnerability, exploit it and obtain a reverse shell. Use **sudo su** in the reverse shell to obtain elevated privileges and find the flag located in the **/root/** directory

- Linux box
- Command injection

nmap:
`sudo nmap -sC -sV -oN vm3 192.168.106.16`

```hlt:7
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ac:d6:d2:e6:6f:60:c2:ed:bc:0e:be:8f:75:e9:37:86 (RSA)
|   256 35:fb:0b:12:ef:d8:f1:f5:d6:82:f9:b9:8d:15:84:00 (ECDSA)
|_  256 5a:28:c2:ae:cd:f0:e6:6c:f4:e3:0c:ad:c4:ff:f7:eb (ED25519)
80/tcp open  http    Werkzeug httpd 2.2.2 (Python 3.9.5)
|_http-title: Intro to Future Factor Authentication
|_http-server-header: Werkzeug/2.2.2 Python/3.9.5
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

![[Pasted image 20250314214559.png]]

"Demo" login page 
![[Pasted image 20250314214747.png]]


Gobuster:
`gobuster dir -u 'http://192.168.106.16/' -w /usr/share/wordlists/dirb/common.txt -t 5`

Only two directories. `/console` and `/login`

![[Pasted image 20250314215009.png]]

Looking at `/console` 
![[Pasted image 20250314215106.png]]
- looks like service account shell will have the pin. initial access first?

Probably need to start burp.

Cheeky curl command revealed a secret?
![[Pasted image 20250314220550.png]]
- `Q2Xf7QbX3XOu4OaRmG4E`
- `curl -X POST --data 'username=ipconfig' http://192.168.106.16/login`

Um
![[Pasted image 20250314220750.png]]

I think the pin is 14 characters.

Used ffuf to get list of valid usernames. 

`ffuf -w /usr/share/seclists/Usernames/top-usernames-shortlist.txt -X POST -d "username=FUZZ&&password=x&&ffa=y" -H "Content-Type: application/x-www-form-urlencoded" -u http://192.168.245.16/login -mr "Invalid Credentials. Please try again."`

![[Pasted image 20250315141856.png]]
Is this the ffa?
![[Pasted image 20250315142107.png]]

**hints**
1. Visit VM #3's webserver to find three fields on the login page.
2. Test various inputs like && to identify the command injection vulnerability.
3. Assume the back-end uses a vulnerable function like **eval()** or **popen()**: **popen(f'echo "test"')**

**Capstone Lab**: Enumerate the machine VM #4. Find the web application and get access to the system. The flag can be found in **C:\inetpub\**.

`feroxbuster --url 'http://192.168.245.192' -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt `

