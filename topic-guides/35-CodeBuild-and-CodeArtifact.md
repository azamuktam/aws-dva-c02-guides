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

**Phase order: `install` → `pre_build` → `build` → `post_build`.**

The order is not arbitrary — it reflects the natural build sequence: get your tools, prepare (log in, fetch dependencies), do the work, publish the result.

- **`runtime-versions` belongs only in the `install` phase.**
- **`finally`** inside a phase runs even when the `commands` block fails — for cleanup and diagnostics.
- If a phase fails, later phases are skipped, with one exception: **`post_build` still runs when `build` fails**, so you can publish test reports from a failed build.

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
- **Never put secrets in `env/variables`**
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

## AWS CodeArtifact

**Why it exists**

Applications depend on packages. Two problems follow: internal libraries need somewhere private to live, and depending directly on public registries means a build breaks whenever npmjs or PyPI has an outage or a package is unpublished. CodeArtifact is a managed package repository that solves both.

- **Formats:** npm, PyPI (pip/twine), Maven, Gradle, NuGet, SwiftPM, Cargo, generic.
- **Structure:** a **Domain** contains **Repositories**, sharing storage and KMS encryption across them.
- **Upstream repositories and external connections** let one repository proxy and **cache** npmjs, PyPI, or Maven Central. Once a package is cached, your builds keep working even if the public registry is unavailable — and you get a record of exactly which versions you consumed.
- **Authentication:** `aws codeartifact get-authorization-token` returns a token valid up to **12 hours**; `aws codeartifact login` configures your package manager for you.
- **Resource policies** grant cross-account access, so several teams can share internal libraries.
- Emits **EventBridge events** on package changes, so publishing a new version can trigger a pipeline.
- Slots into CodeBuild as the dependency source during the `install` phase.

**Recall check — CodeArtifact**

- **Private npm / pip / Maven / NuGet repository** → CodeArtifact
- **Share internal libraries between teams or accounts** → CodeArtifact + resource policy
- **Cache and proxy public packages** → external connection / upstream repository
- **Approved dependencies only** → CodeArtifact as the single configured registry
- **Trigger a build when a package version is published** → CodeArtifact + EventBridge
- **Authentication token lifetime** → up to 12 hours

# Question patterns

> A fully managed CI service that compiles code, runs tests, and produces artifacts with no build servers to manage. Which service?
> → **AWS CodeBuild**

> Where is the CodeBuild build/test configuration defined?
> → **buildspec.yml**

> Where must buildspec.yml live?
> → **At the root of the source directory**

> Can a CodeBuild project override the buildspec location or supply an inline spec?
> → **Yes**

> Where does a CodeBuild build actually run?
> → **In a managed, ephemeral container provisioned per build**

> What are the CodeBuild build phases in order?
> → **install → pre_build → build → post_build**

> In which phase do you set runtime-versions?
> → **install**

> In which phase do you install dependencies?
> → **install**

> In which phase would you log in to ECR?
> → **pre_build**

> In which phase do you compile and test?
> → **build**

> In which phase do you push the image or upload artifacts?
> → **post_build**

> Which block runs even when the phase's commands fail?
> → **finally**

> If the build phase fails, which later phase still runs?
> → **post_build (so you can publish test reports from a failed build)**

> Password to fetch a secret for a build. Which section do you use?
> → **env/secrets-manager**

> Which buildspec section pulls values from SSM Parameter Store?
> → **env/parameter-store**

> Why must secrets never go in env/variables?
> → **It is plaintext, visible in build logs and the console**

> A build cannot read a secret even though the syntax looks right. What should you check?
> → **The CodeBuild service role permissions (ssm:GetParameters / secretsmanager:GetSecretValue, plus kms:Decrypt for CMKs)**

> Which IAM permission fetches from Secrets Manager?
> → **secretsmanager:GetSecretValue**

> Which IAM permission fetches from SSM Parameter Store?
> → **ssm:GetParameters**

> Which extra permission is needed when a secret is encrypted with a customer managed KMS key?
> → **kms:Decrypt**

> Pass a value from the build to a later pipeline stage. Which section?
> → **env/exported-variables**

> Which buildspec section configures test reports, e.g. JUnit XML?
> → **reports**

> Which file format value is used for JUnit test reports?
> → **JUNITXML**

> Which buildspec section declares what the build produces?
> → **artifacts**

> Where do CodeBuild build artifacts go?
> → **S3, or container images to ECR**

> Can CodeBuild encrypt its output artifacts?
> → **Yes, using AWS KMS**

> What is the default CodeBuild timeout?
> → **60 minutes**

> What is the maximum CodeBuild timeout?
> → **8 hours**

> Builds are slow and keep redownloading dependencies. What is the fix?
> → **Configure cache (S3 cache or local cache modes)**

> Which local cache modes are available?
> → **LOCAL_DOCKER_LAYER_CACHE, LOCAL_SOURCE_CACHE, LOCAL_CUSTOM_CACHE**

> What must be enabled to build Docker images?
> → **Privileged mode**

> A build must reach a private RDS instance for integration tests. What do you configure?
> → **Run CodeBuild inside the VPC**

> How can you reproduce a CodeBuild build on your own machine?
> → **The CodeBuild agent (codebuild_build.sh)**

> What shows pass/fail status in a README?
> → **Build badges**

> Give an example of a CodeBuild automatic environment variable.
> → **CODEBUILD_BUILD_ID, CODEBUILD_RESOLVED_SOURCE_VERSION, CODEBUILD_SRC_DIR, or AWS_DEFAULT_REGION**

> Can you SSH into a CodeBuild container?
> → **No — containers are deleted at the end of execution (success or failure)**

> Does a CodeBuild container still exist after the build finishes?
> → **No, it is deleted at the end of execution**

> A static website is generated from Markdown and deployed to S3. Which combination?
> → **CodePipeline + CodeBuild + Amazon S3**

> In that pipeline, what does CodeBuild do?
> → **Runs any command to convert Markdown into static website files**

> A managed private package repository for internal libraries. Which service?
> → **AWS CodeArtifact**

> Which package formats does CodeArtifact support?
> → **npm, PyPI, Maven, Gradle, NuGet, SwiftPM, Cargo, generic**

> In CodeArtifact, what contains repositories and shares storage and KMS encryption across them?
> → **A domain**

> How does CodeArtifact cache and proxy npmjs, PyPI, or Maven Central?
> → **Via upstream repositories and external connections**

> Why cache public packages in CodeArtifact?
> → **Builds keep working if the public registry is down, and you get a record of versions consumed**

> Which command returns a CodeArtifact authorization token, and how long is it valid?
> → **aws codeartifact get-authorization-token — valid up to 12 hours**

> Which command configures your package manager automatically?
> → **aws codeartifact login**

> Several teams must share internal libraries across accounts. What do you use?
> → **CodeArtifact + resource policies**

> How can publishing a new package version trigger a pipeline?
> → **CodeArtifact EventBridge events**

> Where does CodeArtifact slot into CodeBuild?
> → **As the dependency source during the install phase**

> What is the CodeArtifact authorization token maximum lifetime?
> → **Up to 12 hours**

# Pocket card

| Keyword | Answer |
| --- | --- |
| Build config | **buildspec.yml** |
| buildspec location | **Source root** |
| Phase order | **install → pre_build → build → post_build** |
| runtime-versions | **install phase only** |
| Install dependencies | **install** |
| Log in to ECR | **pre_build** |
| Compile and test | **build** |
| Push image / upload | **post_build** |
| Runs on failure | **finally** |
| Runs if build fails | **post_build** |
| Secret from Secrets Manager | **env/secrets-manager** |
| Secret from SSM | **env/parameter-store** |
| Never secrets here | **env/variables (plaintext)** |
| Secret read failure | **CodeBuild service role** |
| Secrets Manager permission | **secretsmanager:GetSecretValue** |
| SSM permission | **ssm:GetParameters** |
| CMK permission | **kms:Decrypt** |
| Pass value to next stage | **env/exported-variables** |
| Test reports | **reports section** |
| JUnit format | **JUNITXML** |
| Output declaration | **artifacts section** |
| Where artifacts go | **S3 (or images to ECR)** |
| Artifact encryption | **AWS KMS** |
| Default timeout | **60 minutes** |
| Max timeout | **8 hours** |
| Slow builds | **Configure cache** |
| Cache modes | **LOCAL_DOCKER_LAYER_CACHE, LOCAL_SOURCE_CACHE, LOCAL_CUSTOM_CACHE** |
| S3 cache | **Remote build cache option** |
| Build Docker images | **Privileged mode** |
| Reach private RDS | **Run CodeBuild in the VPC** |
| Local build | **CodeBuild agent (codebuild_build.sh)** |
| Build status in README | **Build badges** |
| Auto var: build id | **CODEBUILD_BUILD_ID** |
| Auto var: commit | **CODEBUILD_RESOLVED_SOURCE_VERSION** |
| Auto var: source dir | **CODEBUILD_SRC_DIR** |
| Auto var: region | **AWS_DEFAULT_REGION** |
| Container after build | **Deleted (no SSH)** |
| Static site pipeline | **CodePipeline + CodeBuild + S3** |
| Avoid managing Jenkins | **CodeBuild** |
| Private package repo | **CodeArtifact** |
| Package formats | **npm, PyPI, Maven, Gradle, NuGet, SwiftPM, Cargo, generic** |
| Domain contains | **Repositories (shared storage + KMS)** |
| Cache public packages | **Upstream repository / external connection** |
| Auth token lifetime | **Up to 12 hours** |
| Get token | **aws codeartifact get-authorization-token** |
| Configure package manager | **aws codeartifact login** |
| Cross-account library sharing | **Resource policies** |
| Package published event | **EventBridge** |
| CodeBuild dependency source | **CodeArtifact in install phase** |
| Approved dependencies only | **CodeArtifact as the single registry** |
| Managed CI service | **CodeBuild** |
| Buildspec supplies | **Commands to run + artifacts to produce** |
