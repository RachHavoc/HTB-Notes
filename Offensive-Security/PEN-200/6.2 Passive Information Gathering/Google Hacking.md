Using search strings and operators to refine google searches. 

Operators 

`site:` - limits search to single domain
`filetype:` - limit search to specific filetype 
`ext:` - determine which programming languages may be used on a site. e.g. `ext:php, ext:xml, ext:py` 

Can also use `-` to exclude specific values. 
e.g. 
`-filetype:html` - will return only non-HTML pages

`intitle:"index of" "parent directory"` - find pages that contain "index of" in the title and the words "parent directory" i.e. directory listings which may reveal sensitive info

Reference for more dorks:

[Google Hacking DataBase](https://www.exploit-db.com/google-hacking-database)

[DorkSearch](https://dorksearch.com)

Searching for VP of Legal for MegaCorp One

`site: MegaCorpOne | intext: "VP of Legal" intext:NAME intext: EMAIL`
