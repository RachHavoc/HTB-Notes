Framework:
(OWASP, 2022), [https://owasp.org/www-project-top-ten/](https://owasp.org/www-project-top-ten/)

Tools:
* Nmap
* Wappalyzer
* Gobuster
* Burp Suite

### Fingerprinting Web Servers with Nmap 

```shell
kali@kali:~$ sudo nmap -p80  -sV 192.168.50.20
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-29 05:13 EDT
Nmap scan report for 192.168.50.20
Host is up (0.11s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

Apache version 2.4.41 is running on the Ubuntu host

Running Nmap NSE http enumeration script against the target

```
kali@kali:~$ sudo nmap -p80 --script=http-enum 192.168.50.20
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-29 06:30 EDT
Nmap scan report for 192.168.50.20
Host is up (0.10s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum:
|   /login.php: Possible admin folder
|   /db/: BlogWorx Database
|   /css/: Potentially interesting directory w/ listing on 'apache/2.4.41 (ubuntu)'
|   /db/: Potentially interesting directory w/ listing on 'apache/2.4.41 (ubuntu)'
|   /images/: Potentially interesting directory w/ listing on 'apache/2.4.41 (ubuntu)'
|   /js/: Potentially interesting directory w/ listing on 'apache/2.4.41 (ubuntu)'
|_  /uploads/: Potentially interesting directory w/ listing on 'apache/2.4.41 (ubuntu)'

Nmap done: 1 IP address (1 host up) scanned in 16.82 seconds
```

### Wappalyzer: Tech Stack Identification

Perform a Technology Lookup on the [**megacorpone.com** ](https://www.wappalyzer.com/lookup/megacorpone.com/)domain.

Get OS, UI framework, web server, version of javascript libraries etc. 

### GoBuster: Directory Brute Forcing

Running GoBuster using the `dir` mode and `-u` to specify URL `-w` for wordlist

`gobuster dir -u 192.168.50.20 -w /usr/share/wordlists/dirb/common.txt -t 5`

Use `-t 5` to specify number of threads.
Default is 10.


### BurpSuite

Launch from command line 

`kali@kali:~$ burpsuite`

Set up a proxy with `Proxy` tab.

Default proxy listener is `localhost:8080`

I'm going to use FoxyProxy instead of the Mozilla Firefox proxy thing. 

Use Proxy to intercept requests. 

Use Repeater to craft/modify requests

To do this:

 Right-click a request from _Proxy_ > _HTTP History_ and select _Send to Repeater_

Use Intruder for more complex web attacks such as password brute forcing. 


Intercept a login page request with burp

![Figure 14: Simulating a failed WordPress login](https://static.offsec.com/offsec-courses/PEN-200/imgs/webintro/feccd244d6f4b88c7cacd650f713d80a-burp09.png)

Navigate to _Proxy_ > _HTTP History_, right-click on the POST request to **/wp-login.php** and select _Send to Intruder_

![Figure 15: Sending the POST request to Intruder](https://static.offsec.com/offsec-courses/PEN-200/imgs/webintro/5b81caf0153e8c312942e52703825a8a-burp10new.png)

Select Intruder tab. Choose POST request to modify. Move to Positions sub-tab. 

To bruteforce password field:

Select _Clear_ then select value of _pwd_ and press _Add_

![Figure 15: Assigning the password value to the Intruder payload generator](https://static.offsec.com/offsec-courses/PEN-200/imgs/webintro/1377199f64563593c0d812be0dd27fee-burp11.png)

Before starting attack, provide wordlist. 

Providing first 10 values from `rockyou.txt`

```
kali@kali:~$ cat /usr/share/wordlists/rockyou.txt | head
123456
12345
123456789
password
iloveyou
princess
1234567
rockyou
12345678
abc123
```

Paste these value in the _Payload Options_ under _Payloads_ sub-tab

![Figure 15: Pasting the first 10 rockyou entries](https://static.offsec.com/offsec-courses/PEN-200/imgs/webintro/9782513cea865bbb817f691c9354cf57-burp12.png)

Click start attack. 

The different status code `302` may indicate a discovered password. 

![Figure 15: Inspecting Intruder's attack results](https://static.offsec.com/offsec-courses/PEN-200/imgs/webintro/a3888c57e423a1d424c3ce5203b3bc6b-burp13.png)

Confirmed by logging into GUI. 

### Web App Enumeration



