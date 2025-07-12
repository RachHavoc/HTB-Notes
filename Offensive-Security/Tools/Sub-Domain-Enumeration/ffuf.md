Website fuzzing tool

GitHub:
https://github.com/ffuf/ffuf

Example usage:

`ffuf -k -u https://watch.streamio.htb/search.php -d "q=FUZZ" -w /opt/SecLists/Fuzzing/special-chars.txt -H 'Content-Type: application/x-www-form-urlencoded'

