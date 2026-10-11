## AWS CodeBuild and `buildspec.yml`
**AWS CodeBuild:** A fully managed **CI service** that compiles code, runs tests, and produces deployable artifacts. can run any command
**Exam memory:** **CodeBuild = CI → Build + Test + Package.** 
**buildspec.yml:** A YAML file that tells CodeBuild **which commands to run and which artifacts to produce**, usually stored in the source code root.
**Exam memory:** **CodeBuild = runs the build; buildspec.yml = instructions for the build.**
**Why it exists**
Building code needs a machine with the right toolchain. Maintaining Jenkins servers for that is overhead that scales badly and sits idle most of the time. CodeBuild provisions a container per build, runs your commands, produces artifacts, and disappears.
```plain text
Source  →  CodeBuild container  →  runs buildspec.yml commands  →  artifacts → S3 / ECR
```
**`buildspec.yml`**
Lives at the **root of the source directory**, named `buildspec.yml` (the project can override the path or supply an inline spec).
```yaml
version:0.2

env:
variables:                      # plaintext — NEVER secrets
NODE_ENV: production
parameter-store:                # from SSM Parameter Store
DB_HOST: /myapp/prod/db-host
secrets-manager:                # from Secrets Manager
DB_PASSWORD: prod/db:password
exported-variables:             # pass values to later pipeline stages
- IMAGE_TAG

phases:
install:
runtime-versions:
nodejs:20
commands:
- npm ci
pre_build:
commands:
- aws ecr get-login-password | docker login --username AWS --password-stdin $REPO
build:
commands:
- npm test
- docker build -t $REPO:$IMAGE_TAG .
post_build:
commands:
- docker push $REPO:$IMAGE_TAG
finally:
- echo "runs even if this phase failed"

reports:
unit-tests:
files:
-'junit.xml'
file-format: JUNITXML

artifacts:
files:
-'**/*'
base-directory: dist
discard-paths:no

cache:
paths:
-'node_modules/**/*'
```
**Phase order: ****`install`**** → ****`pre_build`**** → ****`build`**** → ****`post_build`****.**
The order is not arbitrary — it reflects the natural build sequence: get your tools, prepare (log in, fetch dependencies), do the work, publish the result.
- **`runtime-versions`**** belongs only in the ****`install`**** phase.**
- **`finally`** inside a phase runs even when the `commands` block fails — for cleanup and diagnostics.
- If a phase fails, later phases are skipped, with one exception: **`post_build`**** still runs when ****`build`**** fails**, so you can publish test reports from a failed build.
**Secrets in builds**
Use **`env/parameter-store`** or **`env/secrets-manager`**. Never `env/variables`, which is plaintext and visible in build logs and the console.
The build’s **service role** needs the matching permissions: `ssm:GetParameters` or `secretsmanager:GetSecretValue`, plus `kms:Decrypt` when the value is encrypted with a customer managed key. A build that fails to fetch a secret is usually a service-role problem, not a syntax problem.
**Other CodeBuild details**
- **Timeout:** default **60 minutes**, maximum **8 hours**. A build that hangs and then fails is usually hitting this — the fix is to raise the timeout or find the hang.
- In `buildspec.yml`, you configure the **`artifacts`** section. CodeBuild can encrypt the output artifacts using **AWS KMS**.
- **Caching:** S3 cache, or local cache modes (`LOCAL_DOCKER_LAYER_CACHE`, `LOCAL_SOURCE_CACHE`, `LOCAL_CUSTOM_CACHE`). This is the answer to “builds are slow and keep redownloading dependencies”.
- **Privileged mode** must be enabled to build Docker images.
- **VPC configuration** lets a build reach private resources — an RDS instance for integration tests, or a private package repository.
- **Local builds** with the CodeBuild agent (`codebuild_build.sh`) let you reproduce a build on your machine.
- **Build badges** show pass/fail status in a README.
- Automatic environment variables: `CODEBUILD_BUILD_ID`, `CODEBUILD_RESOLVED_SOURCE_VERSION`, `CODEBUILD_SRC_DIR`, `AWS_DEFAULT_REGION`.
- Output artifacts go to **S3**, or container images to **ECR**.
- CodeBuild containers are **deleted at the end of their execution** (success or failure). You can't SSH into them, even while they're running.
**Recall check — CodeBuild**
- **Compile code, run unit tests, produce artifacts, no build servers to manage** → CodeBuild
- **`buildspec.yml`** → CodeBuild, at the source root
- **Install dependencies** → `install` phase
- **Set the runtime version** → `runtime-versions`, in `install`
- **Log in to ECR** → `pre_build`
- **Compile and test** → `build`
- **Push the image / upload artifacts** → `post_build`
- **Run cleanup even on failure** → `finally`
- **Pass a database password to a build** → `env/secrets-manager` (or `parameter-store`)
- **Never put secrets in ****`env/variables`**
- **Build can’t read a secret** → check the CodeBuild **service role**
- **Builds are slow / redownload dependencies** → `cache`
- **Build must reach a private RDS instance** → run CodeBuild in the VPC
- **Build running too long / timing out** → default 60 min, max 8 h
- **Build Docker images** → privileged mode
- **Pass a value to the next pipeline stage** → `exported-variables`
- **Avoid managing Jenkins** → CodeBuild
- **Run a build on your own machine** → CodeBuild agent
- **Use:** **CodePipeline + CodeBuild + Amazon S3** — CodeBuild (can run any command) converts the Markdown files into static website files, and CodePipeline automatically deploys the build output to the S3 bucket. 
---
