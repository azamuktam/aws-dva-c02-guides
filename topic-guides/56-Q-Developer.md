### Amazon Q Developer
Added as an explicit new skill in the December 2024 exam guide revision. Expect one or two conceptual questions, not depth.
**What it is.** An AI assistant for software development, embedded in IDEs, the command line, the AWS Console, and chat platforms.
**What it does:**
- **Inline code suggestions** and completions as you type
- **Chat** about your code, AWS services, and your own account’s resources
- **Agentic tasks** — implement a feature across multiple files, write tests, generate documentation
- **Code transformation** — upgrade Java versions, port .NET applications to Linux
- **Security scanning** of your source code during development, with suggested fixes
- **Console and CLI assistance** — explain an error, suggest a command, troubleshoot a resource
**Where it runs:** VS Code, JetBrains, Visual Studio, Eclipse, the CLI, the AWS Console, Slack and Teams.
**The distinction that will be tested:** Q Developer scans **your source code as you write it**. **Amazon Inspector** scans **deployed artifacts** — EC2 instances, container images, Lambda functions — for known CVEs. Different stage of the lifecycle, different service.
**Recall check**
- **AI code suggestions in the IDE** → Amazon Q Developer
- **Automatically upgrade a legacy Java application** → Q Developer code transformation
- **Find security issues in source code during development** → Q Developer security scan
- **Find CVEs in deployed EC2 / containers / Lambda** → **Amazon Inspector** (different service)
- **Explain an AWS error in the console** → Q Developer
---
