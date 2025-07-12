A tool to quickly bruteforce and enumerate valid Active Directory accounts through Kerberos Pre-Authentication

GitHub:

https://github.com/ropnop/kerbrute

Find latest release:

https://github.com/ropnop/kerbrute/releases/tag/v1.0.3

Used in [[Blackfield]]

`kerbrute -h` to list available commands 

`./kerbrute userenum --dc 10.10.10.192 -d blackfield -o kerbrute.username.out users.lst`