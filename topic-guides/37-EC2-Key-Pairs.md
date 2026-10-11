## EC2 Key Pairs Across Regions
**Key point**
> **EC2 key pairs are Regional.** A key pair created in one Region cannot directly be used in another Region.
**Reuse the same SSH key across Regions**
```plain text
Same private key
      │
      └── Same public key
            ├── Import → Region A
            ├── Import → Region B
            └── Import → Region C
```
- Keep the **private key** securely.
- Import the corresponding **public key** into each AWS Region.
- AWS stores the public key; **AWS does not store your private key**.
- If the private key is lost, AWS **cannot recover it**.
**Exam memory**
> **EC2 key pair = Regional**
> **Same key across Regions → import the same public key into each Region**
### SAML = Security Assertion Markup Language
It is a **standard for letting one system prove a user's identity to another system**.
Simple example:
```plain text
You work at a company
      ↓
Company login system
      ↓
SAML message
      ↓
AWS
      ↓
You get access
```
So you **log in once with your company's account**, and AWS trusts the company's identity system.
**Real example**
You open AWS and click:
> **Sign in with Company Account**
Your company identity provider says:
> “This is John, and he is allowed to access AWS.”
It sends a **SAML assertion** to AWS.
AWS then gives John access, usually through an IAM role.
### Important terms
**Identity Provider (IdP)**
→ The system that knows who you are, such as your company's login system.
**SAML assertion**
→ The message saying who you are and what access you should get.
**AWS**
→ Trusts the assertion and gives you access.
### Exam memory
> **SAML = standard used for federated login / SSO**
And your previous question:
**LDAP doesn't support SAML → use a custom Identity Broker + STS.**
