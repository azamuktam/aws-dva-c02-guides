### AWS Systems Manager
**Why it exists.** A collection of operational tools for managing resources at scale. For this exam, three parts matter:
- **Parameter Store** — configuration and secrets (covered fully in Part 4).
- **AppConfig** — feature flags and safe configuration deployment (covered fully in Part 4).
- **Run Command** — execute commands on running instances without SSH. The counterpart to EC2 user data: **user data runs at launch; Run Command manages instances that are already running.**
Also present: Session Manager (browser shell to an instance, no SSH keys or open ports), Patch Manager, State Manager, Automation runbooks.
**Recall check**
- **Manage already-running EC2 instances** → Systems Manager Run Command
- **Configuration values and secrets** → Parameter Store
- **Feature flags with safe rollout** → AppConfig
- **Shell into an instance without SSH keys or open ports** → Session Manager
---
