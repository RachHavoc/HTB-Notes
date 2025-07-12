Password cracking tool

GitHub: https://github.com/vanhauser-thc/thc-hydra

Releases: https://github.com/vanhauser-thc/thc-hydra/releases

Basic usage:

```
hydra -l admin -p password ftp://localhost/
hydra -L default_logins.txt -p test ftp://localhost/
hydra -l admin -P common_passwords.txt ftp://localhost/
hydra -L logins.txt -P passwords.txt ftp://localhost/
```
