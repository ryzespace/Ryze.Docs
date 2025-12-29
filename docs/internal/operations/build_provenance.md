# Build Provenance Documentation

## Overview

This document provides complete, auditable **build provenance information** for all RyzeSpace microservices.

**Microservices included:**
- Ryze.Payment
- Ryze.AuthKit
- Ryze.Marketplace
- Ryze.Host
- Any other services using the same pipeline

### Purpose
- **Auditability:** Trace artifacts to source code
- **Security:** Ensure artifacts haven't been tampered with
- **Compliance:** Meet regulatory requirements
- **Reproducibility:** Ability to recreate builds from source

### Scope
- GitHub Actions workflow execution
- Multi-platform build (Linux, Windows, macOS)
- Artifact generation and attestation
- Verification procedures

---

## Build Process

### 1. Trigger Events
- Push to `main` branch
- Pull Request targeting `main` branch

### 2. Build Pipeline Stages

#### Stage 1: Code Checkout

```
- name: Checkout code   uses: actions/checkout@v4
```

**Actions:**
Clone repository at commit SHA  
**Output:**
Working directory with source code

---

#### Stage 2: .NET SDK Setup

```
- name: Setup .NET SDK   uses: actions/setup-dotnet@v4   with:     dotnet-version: '10.0.100'
```

**Actions:**
Install .NET SDK (locked to 10.0.100)  
**Output:**
.NET CLI tools available

---

#### Stage 3: Dependency Caching / Restoration

```
dotnet restore [SolutionFile]
```

**Actions:**
Restore dependencies from lock files or cache  
**Output:**
Resolved packages

---

#### Stage 4: Build Solution

```
dotnet build [SolutionFile] --configuration Release --no-restore
```

**Actions:**
Compile all projects  
**Output:**
Binaries in `bin/Release/`

---

#### Stage 5: Publish Application

```
dotnet publish [ProjectFile] \   --configuration Release \   --runtime [runtime] \   --self-contained false \   --output ./publish/[runtime]
```

**Actions:**
Create platform-specific artifacts  
**Output:**
Publishable binaries in `./publish/[runtime]`

**Supported Runtimes:**
- linux-x64
- win-x64
- osx-x64

---

#### Stage 6: Artifact Upload

```
- name: Upload artifact   uses: actions/upload-artifact@v4   with:     name: [artifact-name]     path: ./publish/[runtime]
```

**Actions:**
Upload build output to CI storage

---

#### Stage 7: Build Provenance Attestation

```
- name: Attest build provenance   uses: actions/attest-build-provenance@v3   with:     subject-path: ./publish
```

**Actions:**
Generate SLSA provenance, SHA256 hashes, signed statement

---

## Build Environment

|Platform|OS Version|Runner|
|---|---|---|
|Linux|Ubuntu 22.04|ubuntu-latest|
|Windows|Windows Server 2022|windows-latest|
|macOS|macOS 12 Monterey|macos-latest|

**Software Versions:**

- .NET SDK 10.0.100
- NuGet (built-in)
- Git 2.x (runner default)

**Dependencies:** locked via `.csproj` + `packages.lock.json`

---

## Artifacts

### Artifact Structure

```
[publish]/[runtime]/ ├── [ServiceName].dll ├── [ServiceName].runtimeconfig.json ├── [ServiceName].deps.json ├── Related domain/application/infrastructure DLLs ├── appsettings.json ├── appsettings.Production.json └── [runtime dependencies...]
```

### Artifact Metadata

|Property|Description|Example|
|---|---|---|
|Name|Artifact identifier|payment-service-linux-x64|
|Size|Compressed size|15–25 MB|
|Format|Compression format|ZIP|
|Retention|Storage duration|90 days|
|Download URL|Artifact location|GitHub Actions artifacts|

### Attestation Files
- **Format:** JSON Lines (`.jsonl`)
- **Includes:** SLSA provenance, SHA256 digests, build metadata, signature

---

## Attestation Schema
- **SLSA Provenance v0.2**
- **Signature:** Sigstore (Fulcio + Rekor)
- **Verification:** `gh attestation verify`

---

## Verification Process

1. Download artifact:
```
gh run download {run-id} -n [artifact-name]
```
2. Verify attestation:
```
gh attestation verify [artifact-file] --owner {org} --repo {repo}
```

3. Compare SHA256:
```
jq -r '.subject[].digest.sha256' attestation.jsonl sha256sum [artifact-file]
```

4. Verify source code:
```
git clone https://github.com/{org}/{repo}.git git checkout {commit-sha-from-attestation}
```

5. Reproduce build (optional):
```
dotnet restore [SolutionFile] dotnet build [SolutionFile] --configuration Release dotnet publish [ProjectFile] --configuration Release --runtime [runtime] --output ./publish/[runtime] diff -r ./publish/[runtime] ./verify/
```

---

## Security Considerations

- **Deterministic builds:** source + SDK + package versions locked
- **Non-deterministic:** timestamps, paths, random seeds
- **Attestation signing:** Sigstore, ephemeral certificates, transparency log
- **Artifact integrity:** SHA256 checksums + HTTPS download
- **Dependency locking:** packages.lock.json
- **Workflow security:** minimal permissions, ephemeral runners, no secrets
- **External actions:** pinned to commit SHA

---

## References

- Workflow: `.github/workflows/build.yml`
- Attestation spec: [https://slsa.dev/provenance/v0.2](https://slsa.dev/provenance/v0.2)
- Sigstore: [https://www.sigstore.dev/](https://www.sigstore.dev/)
- GitHub CLI: [https://cli.github.com/](https://cli.github.com/)
---

## Contact
- Issues: `https://github.com/{org}/{repo}/issues`
- Security: `SECURITY.md`
- Build failures: check GitHub Actions logs