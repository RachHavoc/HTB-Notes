**Challenge Lab 1: MEDTECH**: You have been tasked to conduct a penetration test for MEDTECH, a recently formed IoT healthcare startup. Your objective is to find as many vulnerabilities and misconfigurations as possible in order to increase their Active Directory security posture and reduce the attack surface.

![Figure 1: Challenge Scenario](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PWKR-LABS/imgs/challengelab1/950940473d8812f4388e53bb3b33852a-Topology2.png)

**172.16.234.10**

Challenge1 - VM 1 OS Credentials:

**172.16.234.11**

Challenge1 - VM 2 OS Credentials:

**192.168.234.120**

Challenge1 - VM 3 OS Credentials:

**192.168.234.121**

Challenge1 - VM 4 OS Credentials:

**192.168.234.122**

Challenge1 - VM 5 OS Credentials:

**172.16.234.12**

Challenge1 - VM 6 OS Credentials:

**172.16.234.13**

Challenge1 - VM 7 OS Credentials:

**172.16.234.14**

Challenge1 - VM 8 OS Credentials:

**172.16.234.82**

Challenge1 - VM 9 OS Credentials:

**172.16.234.83**

Challenge1 - VM 10 OS Credentials:


#### Nmap against all these 

`sudo nmap -sC -sV -oA nmap/172.txt 172.16.234.10,11,12,13,14`

Interesting. All first 1000 ports in "ignored states"

![[Pasted image 20250310115520.png]]

`sudo nmap -sC -sV -oA nmap/192.txt 192.168.234.120,121,122 `

![[Pasted image 20250310115744.png]]

#### Web app enumeration

1. `192.168.234.120:80` - WEBrick
2. `192.168.234.121:80` - IIS Server (looks more like a HVT)

MedTech site

`192.168.234.121:80`

![[Pasted image 20250310120127.png]]
- ![[Pasted image 20250310120336.png]]
- login page with `.aspx` extension
- ![[Pasted image 20250310120604.png]]
	- Potential Username: Robart Brown

GoBuster Time:

`gobuster dir -u 'http://192.168.234.121' -w /usr/share/dirb/wordlists/common.txt`

![[Pasted image 20250310121719.png]]
- Nothing we can access

Trying null smb enumeration (nothing, but got domain name)
![[Pasted image 20250310125410.png]]




Other site 

![[Pasted image 20250310121233.png]]
- Minimal Mistakes Jekyll Theme 4.24.0

Gobuster for this site too 
`gobuster dir -u 'http://192.168.234.120' -w /usr/share/wordlists/dirb/common.txt`

![[Pasted image 20250310121653.png]]
Windows Server 2022 Build 20348 x64 (name:WEB02) (domain:medtech.com)


```

Currently in a view state decoder rabbithole. Deserialized viewstate base64 value using this site:
https://viewstatedecoder.azurewebsites.net/

Got the "ViewState" value from burp: `/wEPDwUKMjA3MTgxMTM4N2RkL7UlJbQLRVEHtdBd2cHsgmzduFNoWHiXrVGu0cD9+jc=`
```

I think it might be a SQLi thing and I just need to be patient. 

Going to run autorecon against the 192 boxes.

