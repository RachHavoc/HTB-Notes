
[_Whois_](https://www.domaintools.com/support/what-is-whois-information-and-why-is-it-valuable/) is a TCP service, tool, and type of database that can provide information about a domain name, such as the [_name server_](https://www.forbes.com/advisor/business/software/what-is-a-name-server/) and [_registrar_](https://www.cloudflare.com/learning/dns/glossary/what-is-a-domain-name-registrar/).

`whois megacorpone.com -h 192.168.50.251`

Useful info: 
Username & Title
Nameservers 

Assuming we have an IP address, we can also use the **whois** client to perform a reverse lookup and gather more information.

```
kali@kali:~$ whois 38.100.193.70 -h 192.168.50.251
...
NetRange:       38.0.0.0 - 38.255.255.255
CIDR:           38.0.0.0/8
NetName:        COGENT-A
...
OrgName:        PSINet, Inc.
OrgId:          PSI
Address:        2450 N Street NW
City:           Washington
StateProv:      DC
PostalCode:     20037
Country:        US
RegDate:        
Updated:        2015-06-04
...
```