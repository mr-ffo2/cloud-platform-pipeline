# Pulumi State Architecture

## Cloud Platform Pipeline

This document defines how Pulumi infrastructure state will be managed for the Cloud Platform Pipeline.

Pulumi state is a critical part of the infrastructure platform because it records the relationship between the desired infrastructure defined in code and the infrastructure currently managed in AWS.

---

## 1. What Is Pulumi State?

Pulumi state records information about infrastructure resources managed by Pulumi.

Conceptually:

```text
Desired Infrastructure
        |
        | Pulumi TypeScript
        v
     Pulumi
        |
        | compares
        v
Pulumi State
        |
        v
Current AWS Infrastructure
```

Pulumi uses this state to determine what resources need to be created, updated, replaced, or removed.

The state therefore becomes an important operational asset.

---

## 2. State Management Requirements

The state architecture must support:

- Secure storage
- Encryption
- Access control
- Team collaboration
- CI/CD access
- Environment separation
- State history and recovery
- Auditing
- Disaster recovery
- Controlled administrative access

The state backend must never be treated as an ordinary public storage bucket.

---

## 3. State Backend Decision

The platform will use an AWS S3-backed Pulumi state architecture.

Conceptually:

```text
GitHub Actions
      |
      v
Pulumi
      |
      v
AWS S3 State Backend
      |
      v
Pulumi Infrastructure
```

The decision is based on the goals of this project.

The platform is designed to demonstrate production-oriented AWS infrastructure management, including IAM, encryption, storage security, access control, and infrastructure automation.

Using an AWS-backed state system allows these capabilities to become part of the platform architecture.

---

## 4. State Security

The Pulumi state bucket will be treated as sensitive infrastructure.

The planned security controls include:

- S3 Block Public Access
- Encryption at rest
- Versioning
- Restricted IAM permissions
- No public bucket policies
- Controlled CI/CD access
- Monitoring and auditing
- Recovery considerations

The exact implementation will be introduced during the infrastructure security milestones.

---

## 5. Encryption

Pulumi state must be protected against unauthorized access.

The S3 backend will use encryption at rest.

The architecture may initially use AWS-managed encryption and can later be strengthened with a customer-managed AWS KMS key when the platform requires more explicit key-management controls.

Encryption protects stored state but does not replace IAM authorization.

Both controls are required:

```text
Encryption
     +
IAM Access Control
     =
Protected State
```

---

## 6. Access Control

Access to the Pulumi state backend will be controlled through AWS IAM.

The platform will follow least privilege.

Only identities that require infrastructure state access should receive it.

Conceptually:

```text
Developer
    |
    X
    |
Pulumi State

GitHub Actions
    |
    v
IAM Role
    |
    v
Pulumi State
```

Developers should not automatically receive unrestricted direct access to the production state backend.

Infrastructure access should primarily occur through approved workflows and controlled administrative processes.

---

## 7. Environment Separation

Different environments should have separate Pulumi state.

The intended architecture is:

```text
Development
    |
    v
Dev State

Staging
    |
    v
Staging State

Production
    |
    v
Production State
```

This prevents a development workflow from accidentally operating against production infrastructure state.

---

## 8. Bootstrap Architecture

The state backend creates a bootstrap dependency.

Pulumi requires a state backend to reliably manage the platform, while the state backend itself is AWS infrastructure.

Therefore the platform will use a controlled bootstrap process.

Conceptually:

```text
Bootstrap
    |
    v
S3 State Backend
    |
    v
Main Pulumi Platform
    |
    +-- IAM
    +-- Networking
    +-- Security
    +-- Storage
    +-- Monitoring
    +-- Application Infrastructure
```

The bootstrap layer will be kept intentionally small.

Its purpose is to establish the infrastructure required for the main platform to operate safely.

---

## 9. State Versioning

S3 versioning will be enabled for the state backend.

Versioning provides historical versions of objects and can help recover from accidental modification or deletion.

Conceptually:

```text
State v1
State v2
State v3
State v4
```

This is not a replacement for a complete disaster recovery strategy, but it provides an important recovery capability.

---

## 10. CI/CD Integration

GitHub Actions will eventually interact with the Pulumi state backend through AWS IAM.

The authentication model will be:

```text
GitHub Actions
       |
       | OIDC
       v
AWS IAM
       |
       | Temporary credentials
       v
Pulumi
       |
       v
S3 State Backend
```

No permanent AWS access keys will be required for the normal GitHub-to-AWS infrastructure deployment workflow.

---

## 11. State and Secrets

Pulumi state must not be confused with the application's secret-management system.

The platform has separate responsibilities:

```text
Application Secrets
        |
        v
AWS Secrets Manager


Infrastructure State
        |
        v
S3 Pulumi Backend


GitHub Workflow Secrets
        |
        v
GitHub Secrets


AWS Authentication
        |
        v
GitHub OIDC + IAM
```

Each mechanism exists for a different purpose.

---

## 12. State Access and Production

Production infrastructure state is highly sensitive from an operational perspective.

Production access will therefore have stronger controls than development access.

Potential controls include:

- Protected GitHub environments
- Restricted IAM roles
- Production deployment approvals
- Branch protection
- Audit logging
- Limited administrative access
- Separation of duties

The exact controls will be implemented as the CI/CD and IAM architecture develops.

---

## 13. Failure Scenarios

The state architecture considers several failure scenarios.

### Accidental state deletion

Mitigations:

- S3 versioning
- Restricted permissions
- Recovery procedures
- Administrative access controls

### Unauthorized state access

Mitigations:

- IAM
- Encryption
- Block Public Access
- Restricted bucket policies
- Audit logging

### Compromised CI workflow

Mitigations:

- GitHub OIDC
- Restricted IAM trust policy
- Least-privilege permissions
- Protected environments
- Branch protection

### Regional or infrastructure failure

Mitigations to be considered:

- State versioning
- Backup strategy
- Recovery procedures
- Potential cross-region recovery depending on production requirements

---

## 14. Operational Principle

Pulumi state is considered a critical platform dependency.

Changes to the state backend must therefore be treated with the same care as changes to production infrastructure.

The state backend should not be casually deleted, recreated, or manually modified.

---

## 15. Architecture Decision

The platform adopts the following state architecture:

```text
                         GitHub
                            |
                            v
                     GitHub Actions
                            |
                           OIDC
                            |
                            v
                       AWS IAM Role
                            |
                    Temporary Credentials
                            |
                            v
                         Pulumi
                            |
                            v
                  S3 Pulumi State Backend
                            |
                            v
                     AWS Infrastructure
```

The S3 state backend will be secured using encryption, versioning, restricted access, and public-access prevention.

Development, staging, and production will use separated infrastructure state.

A bootstrap process will establish the state backend before the main infrastructure platform is managed through it.

This architecture provides the foundation for secure Pulumi-based infrastructure automation.
