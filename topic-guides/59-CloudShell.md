### AWS CloudShell
**What it is.** A browser-based shell inside the AWS Console, pre-loaded with the **AWS CLI**, Python, Node.js, and Git.
**Why it’s useful:** it runs with **your console identity’s permissions**, so there are no access keys to configure and nothing to install. The fastest path from “I’m signed in” to “I can run a CLI command”.
**The limit worth knowing:** **1 GB of persistent storage per Region**, in `$HOME`. Everything outside `$HOME` is wiped after a period of inactivity.
**Recall check**
- **Run CLI commands with no local setup and no access keys** → CloudShell
- **CloudShell files disappeared** → only `$HOME` (1 GB per Region) persists
---
