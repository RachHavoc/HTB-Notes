[_Domain Name System_](https://www.cloudflare.com/learning/dns/what-is-dns/) (DNS) is a distributed database responsible for translating user-friendly domain names into IP addresses

DNS Records:

* NS: Nameserver contains name of authoritative servers hosting DNS records for domain
* A: contains IPv4 of a hostname
	`host www.megacorpone.com`
* AAAA: contains IPv6 of a hostname
* MX: Mail Exchange records
	`host -t mx megacorpone.com`
* PTR: Pointer records used in reverse lookup zones
* CNAME: Canonical Name Records used to create aliases for other host records
* TXT: Text records contain various data
	`host -t txt megacorpone.com`

Automating DNS-lookup in a bash script.

Create list of possible hostnames:

```
kali@kali:~$ cat list.txt
www
ftp
mail
owa
proxy
router
```

Bash one-liner:

```
kali@kali:~$ for ip in $(cat list.txt); do host $ip.megacorpone.com; done
www.megacorpone.com has address 149.56.244.87
Host ftp.megacorpone.com not found: 3(NXDOMAIN)
mail.megacorpone.com has address 51.222.169.212
Host owa.megacorpone.com not found: 3(NXDOMAIN)
Host proxy.megacorpone.com not found: 3(NXDOMAIN)
router.megacorpone.com has address 51.222.169.214
```

More wordlists here:

 [SecLists](https://github.com/danielmiessler/SecLists)

`sudo apt install seclists`

Installed at `/usr/share/seclists`

Bruteforcing reverse lookups to request the hostname of each IP. Loop through `51.22.169.200-51.222.169.254` Filter out invalid results with `grep -v`

```
kali@kali:~$ for ip in $(seq 200 254); do host 51.222.169.$ip; done | grep -v "not found"
...
208.169.222.51.in-addr.arpa domain name pointer admin.megacorpone.com.
209.169.222.51.in-addr.arpa domain name pointer beta.megacorpone.com.
210.169.222.51.in-addr.arpa domain name pointer fs1.megacorpone.com.
211.169.222.51.in-addr.arpa domain name pointer intranet.megacorpone.com.
212.169.222.51.in-addr.arpa domain name pointer mail.megacorpone.com.
213.169.222.51.in-addr.arpa domain name pointer mail2.megacorpone.com.
214.169.222.51.in-addr.arpa domain name pointer router.megacorpone.com.
215.169.222.51.in-addr.arpa domain name pointer siem.megacorpone.com.
216.169.222.51.in-addr.arpa domain name pointer snmp.megacorpone.com.
217.169.222.51.in-addr.arpa domain name pointer syslog.megacorpone.com.
218.169.222.51.in-addr.arpa domain name pointer support.megacorpone.com.
219.169.222.51.in-addr.arpa domain name pointer test.megacorpone.com.
220.169.222.51.in-addr.arpa domain name pointer vpn.megacorpone.com.
...
```

Tools to automate DNS Enumeration on Kali:

#dnsrecon 
[_DNSRecon_](https://github.com/darkoperator/dnsrecon) is an advanced DNS enumeration script
#dnsenum 
DNSEnum

Executing this against megacorpone.com with `-d` for domain name and `-t std` to perform standard scan 

```
kali@kali:~$ dnsrecon -d megacorpone.com -t std
[*] std: Performing General Enumeration against: megacorpone.com...
[-] DNSSEC is not configured for megacorpone.com
[*] 	 SOA ns1.megacorpone.com 51.79.37.18
[*] 	 NS ns1.megacorpone.com 51.79.37.18
[*] 	 NS ns3.megacorpone.com 66.70.207.180
[*] 	 NS ns2.megacorpone.com 51.222.39.63
[*] 	 MX mail.megacorpone.com 51.222.169.212
[*] 	 MX spool.mail.gandi.net 217.70.178.1
[*] 	 MX fb.mail.gandi.net 217.70.178.217
[*] 	 MX fb.mail.gandi.net 217.70.178.216
[*] 	 MX fb.mail.gandi.net 217.70.178.215
[*] 	 MX mail2.megacorpone.com 51.222.169.213
[*] 	 TXT megacorpone.com Try Harder
[*] 	 TXT megacorpone.com google-site-verification=U7B_b0HNeBtY4qYGQZNsEYXfCJ32hMNV3GtC0wWq5pA
[*] Enumerating SRV Records
[+] 0 Records Found
```

Now bruteforcing additional hostnames using `list.txt` created earlier with 

`-d`- specify domain name
`-D`- specify wordlist
`-t brt`- specify type of enumeration as brute force

```
kali@kali:~$ dnsrecon -d megacorpone.com -D ~/list.txt -t brt
[*] Using the dictionary file: /home/kali/list.txt (provided by user)
[*] brt: Performing host and subdomain brute force against megacorpone.com...
[+] 	 A www.megacorpone.com 149.56.244.87
[+] 	 A mail.megacorpone.com 51.222.169.212
[+] 	 A router.megacorpone.com 51.222.169.214
[+] 3 Records Found
`````

Enumerating with #dnsenum

```
kali@kali:~$ dnsenum megacorpone.com
...
dnsenum VERSION:1.2.6

-----   megacorpone.com   -----

...

Brute forcing with /usr/share/dnsenum/dns.txt:
_______________________________________________

admin.megacorpone.com.                   5        IN    A        51.222.169.208
beta.megacorpone.com.                    5        IN    A        51.222.169.209
fs1.megacorpone.com.                     5        IN    A        51.222.169.210
intranet.megacorpone.com.                5        IN    A        51.222.169.211
mail.megacorpone.com.                    5        IN    A        51.222.169.212
mail2.megacorpone.com.                   5        IN    A        51.222.169.213
ns1.megacorpone.com.                     5        IN    A        51.79.37.18
ns2.megacorpone.com.                     5        IN    A        51.222.39.63
ns3.megacorpone.com.                     5        IN    A        66.70.207.180
router.megacorpone.com.                  5        IN    A        51.222.169.214
siem.megacorpone.com.                    5        IN    A        51.222.169.215
snmp.megacorpone.com.                    5        IN    A        51.222.169.216
syslog.megacorpone.com.                  5        IN    A        51.222.169.217
test.megacorpone.com.                    5        IN    A        51.222.169.219
vpn.megacorpone.com.                     5        IN    A        51.222.169.220
www.megacorpone.com.                     5        IN    A        149.56.244.87
www2.megacorpone.com.                    5        IN    A        149.56.244.87


megacorpone.com class C netranges:
___________________________________

 51.79.37.0/24
 51.222.39.0/24
 51.222.169.0/24
 66.70.207.0/24
 149.56.244.0/24


Performing reverse lookup on 1280 ip addresses:
________________________________________________

18.37.79.51.in-addr.arpa.                86400    IN    PTR      ns1.megacorpone.com.
...
```

Tools to automate DNS Enumeration on Windows:

#nslookup 

```
C:\Users\student>nslookup mail.megacorptwo.com
DNS request timed out.
    timeout was 2 seconds.
Server:  UnKnown
Address:  192.168.50.151

Name:    mail.megacorptwo.com
Address:  192.168.50.154
```

Query for txt records belonging to specific host. 

```
C:\Users\student>nslookup -type=TXT info.megacorptwo.com 192.168.50.151
Server:  UnKnown
Address:  192.168.50.151

info.megacorptwo.com    text =

        "greetings from the TXT record body"
```

DNS enumeration on MX records of megacorpone.com. What is second-to-best priority value listed?

`host -t mx megacorpone.com`

ANS:20 

Number of TXT records associated with  megacorpone.com domain?

`host -t txt megacorpone.com`

ANS: 2

Automate the DNS enumeration of the megacorpone.com domain with _DNSEnum_. What is the IP of the **siem.megacorpone.com** host?

`dnsenum megacorpone.com`

Power up 'Walk Through VM Group 1', connect to the Windows 11 Client (using xfreerdp), and enumerate the DNS _megacorptwo.com_ and its subdomains through nslookup as illustrated in this section. What text is contained within the TXT record of the _info.megacorptwo.com_ domain?

