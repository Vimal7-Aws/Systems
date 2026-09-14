AWS WAF (Web Application Firewall) is a managed firewall service from AWS that protects web applications from common web exploits and bots before they reach your application.

### Key Features

* **Protects against common attacks**

    * SQL Injection (SQLi)
    * Cross-Site Scripting (XSS)
    * Remote File Inclusion
    * HTTP Flood attacks
    * Malicious bots
    * IP-based attacks

* **Rule Types**

    * **Managed Rules**: Prebuilt rule sets maintained by AWS and partners.
    * **Custom Rules**: Create your own rules using IP addresses, headers, query strings, URI paths, HTTP methods, etc.
    * **Rate-based Rules**: Automatically block or throttle IPs sending excessive requests.
    * **Geo Match Rules**: Allow or block traffic from specific countries.

### How AWS WAF Works

```
Internet
    |
    v
+----------------+
| AWS WAF        |
+----------------+
    |
    +-----------------------------+
    |                             |
    v                             v
CloudFront                  Application Load Balancer
                                   |
                                   v
                            EC2 / ECS / EKS / Lambda
```

AWS WAF can be attached to:

* Amazon CloudFront
* Application Load Balancer (ALB)
* Amazon API Gateway
* AWS AppSync
* Amazon Cognito
* AWS Verified Access

### Components

1. **Web ACL (Access Control List)**

    * A collection of rules.
    * Associated with a protected resource.

2. **Rules**

    * Define conditions to inspect requests.
    * Actions:

        * Allow
        * Block
        * Count
        * CAPTCHA
        * Challenge

3. **Rule Groups**

    * Reusable collections of rules.
    * Can be AWS-managed or customer-managed.

### Example

Suppose your application receives requests like:

```
GET /login
```

An attacker sends:

```
GET /login?id=1' OR '1'='1
```

AWS WAF's SQL Injection rule detects the malicious pattern and blocks the request before it reaches your application.

### Common Use Cases

* Block SQL injection and XSS attacks
* Restrict access by country
* Block known malicious IP addresses
* Prevent brute-force login attempts using rate limiting
* Protect REST APIs
* Mitigate Layer 7 (HTTP/HTTPS) DDoS attacks (often used together with AWS Shield)

### AWS WAF vs Security Groups vs NACLs

| Feature               | AWS WAF              | Security Group     | Network ACL        |
| --------------------- | -------------------- | ------------------ | ------------------ |
| OSI Layer             | Layer 7 (HTTP/HTTPS) | Layer 4            | Layer 4            |
| Filters               | Web requests         | IP, Port, Protocol | IP, Port, Protocol |
| SQLi/XSS Protection   | ✅                    | ❌                  | ❌                  |
| URL/Header Inspection | ✅                    | ❌                  | ❌                  |
| Rate Limiting         | ✅                    | ❌                  | ❌                  |
| Stateful              | N/A                  | ✅                  | ❌                  |

### Best Practices

* Enable **AWS Managed Rule Groups** as a baseline.
* Add **rate-based rules** to protect login and API endpoints.
* Use **Count** mode first to monitor the impact of new rules before switching to **Block**.
* Enable logging to CloudWatch or S3 for analysis.
* Combine AWS WAF with **AWS Shield Standard** (included at no extra cost) for enhanced DDoS protection.
* Regularly review logs and tune custom rules to minimize false positives.

AWS WAF is an essential security service for protecting web applications from common attacks while giving you fine-grained control over which HTTP(S) requests are allowed or blocked.
