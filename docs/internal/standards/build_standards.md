# Build Standards

## Purpose

This document defines **organization-wide standards and best practices for building microservices**. It ensures consistency, security, reproducibility, and maintainability across all CI/CD pipelines.

**Goals:**
- Maintain consistent build environment and tooling
- Enforce secure and auditable build practices
- Standardize artifact naming and structure
- Facilitate reproducible builds
- Provide guidance for new microservices

---

## Table of Contents

1. [Build Stack](#build-stack)
2. [Pipeline Conventions](#pipeline-conventions)
3. [Artifact Standards](#artifact-standards)
4. [Dependency Management](#dependency-management)
5. [Security Requirements](#security-requirements)
6. [Attestation & Verification](#attestation--verification)
7. [Cross-Service Notes](#cross-service-notes)
8. [References](#references)

---

## Build Stack

All microservices must use the **approved stack**:

| Component               | Version / Requirement                                | Notes                                     |
| ----------------------- | ---------------------------------------------------- | ----------------------------------------- |
| .NET SDK                | 10.0.100                                             | Locked version, all builds must use this  |
| NuGet / Package Manager | Latest compatible                                    | Packages must be pinned via lock files    |
| OS / Runner             | Ubuntu 22.04, Windows Server 2022, macOS 12 Monterey | Multi-platform support                    |
| Git                     | 2.x (default runner)                                 | For commit SHA verification               |
| CI/CD                   | GitHub Actions                                       | Workflow must follow pipeline conventions |

---

## Pipeline Conventions

**Trigger events:**
- `push` to `main` branch
- `pull_request` targeting `main`

**Pipeline stages (mandatory):**
1. **Checkout code** – pin to exact commit SHA
2. **SDK / environment setup** – lock versions
3. **Dependency caching / restore** – restore from lock files
4. **Build solution / project** – Release configuration
5. **Publish application / package** – platform-specific artifacts
6. **Upload artifacts** – artifact storage in GitHub Actions
7. **Build attestation** – generate SLSA provenance

**Runner settings:**
- Ephemeral runners (destroyed after build)
- Network isolation for security
- Minimum required permissions:
```yaml
permissions:
  contents: read
  id-token: write
  attestations: write
```

**Best Practices:**
- Pin all GitHub Actions to commit SHA
- Cache dependencies where possible
- Avoid storing secrets in build environment
- Include SLSA attestation for all production artifacts

---

## Artifact Standards

| Property    | Requirement / Convention                                              |
| ----------- | --------------------------------------------------------------------- |
| Naming      | `{service-name}-{runtime}` e.g., `payment-service-linux-x64`          |
| Structure   | Consistent directory layout: binaries, configs, runtime dependencies  |
| Format      | ZIP (default GitHub Actions)                                          |
| Retention   | 90 days                                                               |
| Attestation | SLSA Provenance JSON Lines (`.jsonl`)                                 |

---

## Dependency Management

- Lock files **mandatory**:
  - .NET: `packages.lock.json`
  - Python: `requirements.txt` + hash pinning
  - Node.js: `package-lock.json`
- Restore in locked mode only
- Check for drift / dependency changes in pull requests
- Avoid mutable dependencies from public feeds without verification

---

## Security Requirements

- **Attestation Signing:** Sigstore (Fulcio + Rekor)
- **Non-repudiation:** All builds must generate signed attestation
- **Minimal Permissions:** Only required access for workflow
- **Secrets Handling:** Never log secrets, use GitHub Secrets, rotate regularly
- **Deterministic Builds:**
  - Source code pinned to commit SHA
  - SDK and package versions locked
  - Build configuration standardized
  - Optional: reproducible binaries (functional equivalence sufficient)

---

## Attestation & Verification

All artifacts must include **SLSA provenance**:
- `subject` – artifact names + SHA256
- `predicateType` – `https://slsa.dev/provenance/v0.2`
- `builder.id` – GitHub Actions runner
- `invocation.configSource` – source repository + commit SHA
- Signed using Sigstore, logged in Rekor

Verification workflow (centralized reference in Ryze.Docs):
1. Download artifact
2. Verify attestation signature
3. Compare SHA256 checksums
4. Optional: reproduce build

---

## Cross-Service Notes

- **Central BUILD_PROVENANCE.md** in `Ryze.Docs` is the reference for all microservices
- Individual microservices store **artifacts and attestation**, but **no need for separate provenance docs**
- Standard stack ensures **uniformity**: same runtime, dependency policy, attestation format

---

## References

- [SLSA Provenance v0.2](https://slsa.dev/provenance/v0.2)
- [Sigstore / Cosign](https://www.sigstore.dev/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- Internal:
  - `BUILD_PROVENANCE.md` (Ryze.Docs)
  - `DEPLOYMENT.md` (service deployment guide)
  - `SECURITY.md` (organization security policy)

---