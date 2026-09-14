Great question. **AWS WAF is your first line of defense**, but **your application and microservices should never rely on WAF alone**. A production system uses **defense in depth**, where every layer assumes the previous layer could fail.

Here's how you can prevent each attack from an application/microservices perspective.

---

# Typical Request Flow

```text
Internet
      │
      ▼
AWS WAF
      │
      ▼
CloudFront
      │
      ▼
Application Load Balancer
      │
      ▼
Spring Boot Microservice
      │
      ▼
Business Logic
      │
      ▼
Database
```

Even if a malicious request gets past WAF, your application should still reject it.

---

# 1. SQL Injection

## ❌ Bad Practice

Building SQL using string concatenation.

```java
String sql =
    "SELECT * FROM users WHERE username='"
    + username +
    "' AND password='" +
    password + "'";
```

User enters

```
' OR 1=1 --
```

Query becomes

```sql
SELECT *
FROM users
WHERE username=''
OR 1=1
```

Boom.

---

## ✅ Good Practice

Use prepared statements.

```java
PreparedStatement ps =
connection.prepareStatement(
"SELECT * FROM users WHERE username=? AND password=?");

ps.setString(1, username);
ps.setString(2, password);
```

The database treats input purely as **data**, not SQL.

---

### In Spring Boot

Use Spring Data JPA.

```java
userRepository.findByUsername(username);
```

or

```java
@Query("select u from User u where u.username=:username")
```

Avoid concatenating JPQL or SQL strings.

---

### Also

Validate input.

Example

```java
username.matches("[a-zA-Z0-9]{3,20}")
```

Reject invalid characters before querying.

---

# 2. Cross Site Scripting (XSS)

Suppose a comment API

```http
POST /comments
```

receives

```html
<script>alert("Hacked")</script>
```

---

## Prevention

### Escape HTML

Instead of returning

```html
<script>alert()</script>
```

Return

```html
&lt;script&gt;alert()&lt;/script&gt;
```

Browser displays text instead of executing it.

---

### Sanitize Input

Libraries remove dangerous tags.

Allowed

```html
<b>Hello</b>
```

Removed

```html
<script>...</script>
```

---

### Content Security Policy (CSP)

HTTP Header

```
Content-Security-Policy:
default-src 'self'
```

Browser refuses external JavaScript.

---

### HttpOnly Cookies

```
Set-Cookie:
HttpOnly
Secure
SameSite=Lax
```

JavaScript cannot steal session cookies.

---

# 3. Remote File Inclusion

Never do

```java
String file = request.getParameter("file");

Files.readString(Path.of(file));
```

Attacker passes

```
../../etc/passwd
```

or

```
http://evil.com/shell.php
```

---

## Prevention

Whitelist.

Instead of

```
file=userInput
```

Use

```java
Map<String,String>

home -> home.html

about -> about.html
```

Only known files are allowed.

---

Also

Normalize paths.

Reject

```
..
```

Reject

```
http://
```

Reject

```
ftp://
```

Reject

```
file://
```

---

# 4. HTTP Flood

Suppose your login endpoint receives

```
10000 requests/sec
```

---

## Application Protection

### Rate Limiting

Redis example

```
User/IP

↓

Counter

↓

100 requests/minute

↓

429 Too Many Requests
```

Spring libraries

* Bucket4j
* Resilience4j RateLimiter

---

### Circuit Breaker

If downstream DB is overloaded

```
Reject requests quickly

instead of

Waiting 30 seconds
```

---

### Queue

Instead of

```
Request

↓

Email Service
```

Use

```
Request

↓

SQS

↓

Worker
```

Protects backend.

---

### Caching

Frequently requested data

```
Redis
```

instead of

```
Database
```

---

# 5. Malicious Bots

Bots usually

```
Login

Login

Login

Login

```

thousands of times.

---

Application defenses

### CAPTCHA

Before login

```
Too many failures

↓

Show CAPTCHA
```

---

### Account Lock

```
5 failed attempts

↓

Lock for 15 minutes
```

---

### MFA

Even if password leaked

Attacker still needs

```
OTP
```

---

### Device Fingerprinting

Detect

```
Same IP

Same Browser

100 accounts
```

Flag suspicious activity.

---

# 6. IP Based Attacks

Application keeps track of

```
IP

↓

Requests

↓

Failures

↓

Blacklist
```

Example

```
10 login failures

↓

Ban IP

30 minutes
```

---

Geo restrictions

```
Only Australia

↓

Reject other countries
```

(if applicable)

---

# General Security Best Practices for Microservices

## Input Validation

Never trust input.

Use

```java
@NotBlank

@Email

@Size

@Pattern
```

Example

```java
public class UserRequest {

    @Email
    private String email;

    @Size(max=50)
    private String name;
}
```

---

## Authentication

OAuth2

JWT

OIDC

Never expose internal APIs.

---

## Authorization

Don't do

```
User logged in

↓

Allow everything
```

Instead

```
JWT

↓

Role

↓

Permission

↓

Endpoint
```

---

## Secure Headers

Return

```
X-Frame-Options

Content-Security-Policy

Strict-Transport-Security

X-Content-Type-Options
```

Spring Security configures many of these by default.

---

## HTTPS Everywhere

Never expose

```
HTTP
```

Use

```
HTTPS
```

TLS 1.2+

---

## Secrets

Don't store

```java
password=admin123
```

inside

```
application.yml
```

Use

* AWS Secrets Manager
* AWS Systems Manager Parameter Store
* Kubernetes Secrets (preferably backed by a secret manager)

---

## Logging

Log

```
Who

When

IP

Endpoint

Response
```

But never log

* Passwords
* JWT tokens
* Credit card numbers
* Personally identifiable information (unless required and properly protected)

---

## Dependency Scanning

Regularly update libraries.

Watch for CVEs using tools like

* OWASP Dependency-Check
* Snyk
* Dependabot
* Trivy (for container images)

---

# In a Production Spring Boot Microservices Architecture

```text
                 Internet
                     │
                     ▼
                AWS Shield
                     │
                     ▼
                 AWS WAF
                     │
                     ▼
               CloudFront
                     │
                     ▼
                    ALB
                     │
                     ▼
              API Gateway (optional)
                     │
                     ▼
        Spring Boot Microservices
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
 Authentication  Validation   Authorization
        │            │            │
        └────────────┼────────────┘
                     ▼
              Business Logic
                     │
                     ▼
           Database (Parameterized Queries)
                     │
                     ▼
             Audit Logs / Monitoring
```

The key principle is that **every layer provides protection**:

* **AWS WAF** filters common web attacks before they reach your infrastructure.
* **Spring Security** enforces authentication, authorization, CSRF protection (where applicable), secure headers, and session/token security.
* **Application code** validates and sanitizes input and uses safe APIs like prepared statements.
* **The database layer** uses parameterized queries and least-privilege credentials.
* **Infrastructure** (Security Groups, IAM, Secrets Manager, monitoring) limits the impact even if an attacker gets through one layer.

This layered approach ensures there is **no single point of failure** in your application's security.
