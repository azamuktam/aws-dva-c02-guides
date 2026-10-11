### AWS Amplify
**What it is.** A full-stack platform for web and mobile applications: **hosting + CI/CD + a generated backend**.
Backend resources it provisions for you: **Cognito** (auth), **AppSync/DynamoDB** (data), **S3** (storage), Lambda (functions).
**Amplify Hosting** — connect a Git branch and get automatic build and deploy, **pull-request preview environments**, custom domains, atomic deployments, and password protection.
**When Amplify vs S3 + CloudFront:** Amplify when you want Git-based CI/CD and a generated backend. S3 + CloudFront when you only need static file serving and want full control of caching and distribution behaviour.
**Recall check**
- **Host a React/Vue front end with Git-based CI/CD** → Amplify Hosting
- **Add authentication to a mobile app quickly** → Amplify (Cognito underneath)
- **PR preview environments** → Amplify Hosting
- **Static files only, full control of caching** → S3 + CloudFront
---
