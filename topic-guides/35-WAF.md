## AWS WAF
**Why it exists**
Security groups and NACLs filter by IP address and port — they cannot tell a legitimate request from a SQL-injection payload, because both arrive on port 443. WAF inspects the **HTTP layer** and filters on request content.
- Protects against **SQL injection**, **cross-site scripting (XSS)**, bad bots, and malicious IP ranges.
- **Rate-based rules** throttle a source IP that exceeds a request threshold — basic protection against brute force and layer-7 floods.
- **Managed rule groups** from AWS and vendors cover common threats without writing rules.
- Attaches to **CloudFront, Application Load Balancer, API Gateway**, AppSync, and Cognito user pools.
WAF blocks requests before they reach your application, so the attack never consumes a Lambda invocation or a database connection.
**Recall check — WAF**
- **Block malicious web requests / SQL injection / XSS** → WAF
- **Filter by IP and port only** → security group or NACL, not WAF
- **Throttle a single abusive IP at layer 7** → WAF rate-based rule
- **DDoS protection** → Shield (Standard is automatic; Advanced is a paid add-on)
---
