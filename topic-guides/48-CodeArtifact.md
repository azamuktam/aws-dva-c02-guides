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
