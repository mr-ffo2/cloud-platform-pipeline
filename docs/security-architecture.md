# Security Architecture

## Cloud Platform Pipeline

This document defines the security architecture for the Cloud Platform Pipeline.

The platform is designed around AWS, Pulumi, TypeScript, GitHub Actions, and infrastructure-as-code principles.

The objective is to create a cloud platform that is secure by design, auditable, maintainable, and capable of supporting multiple environments as the platform grows.

---

## 1. Security Objectives

The platform follows these primary security objectives:

1. Avoid long-lived cloud credentials.
2. Apply least-privilege access.
3. Separate authentication from authorization.
4. Keep application secrets outside source control.
5. Separate development, staging, and production environments.
6. Make infrastructure changes auditable.
7. Prevent accidental exposure of sensitive information.
8. Reduce the blast radius of compromised identities.
9. Support secret rotation.
10. Make security controls reproducible through infrastructure-as-code.

Security is treated as part of the platform architecture rather than as a final deployment step.

---

## 2. Security Architecture

The high-level security flow is:

```text
Developer
    |
    v
GitHub
    |
    | GitHub Actions
    |
    | OIDC authentication
    v
AWS IAM Role
    |
    | Short-lived AWS credentials
    |
    v
AWS Infrastructure
    |
    +----------------------+
    |                      |
    v                      v
AWS Services        AWS Secrets Manager
                         |
                         +-- Database credentials
                         +-- API keys
                         +-- OAuth secrets
                         +-- Application secrets
```

The architecture deliberately avoids storing long-lived AWS access keys inside GitHub Actions.

---

## 3. Authentication

Authentication answers:

> Who are you?

GitHub Actions will authenticate to AWS using OpenID Connect (OIDC).

GitHub OIDC is the preferred mechanism for authenticating GitHub Actions workloads to AWS, including both infrastructure deployments and application deployments that require AWS access.

The GitHub workflow will obtain an OIDC identity token and exchange it through AWS IAM for temporary credentials associated with a specific IAM role.

This removes the need to store permanent AWS access keys in GitHub repository secrets.

### Authentication flow

```text
GitHub Actions
      |
      | OIDC token
      v
AWS IAM OIDC Provider
      |
      | Trust policy validation
      v
IAM Deployment Role
      |
      | Temporary credentials
      v
AWS
```

The IAM trust policy will restrict which GitHub repository, branch, or environment is allowed to assume the role.

---

## 4. Authorization

Authorization answers:

> What are you allowed to do?

AWS IAM will control authorization.

Authentication alone does not give GitHub permission to perform arbitrary AWS operations.

The IAM role assumed by GitHub Actions will contain policies defining the AWS resources and actions the workflow is allowed to access.

The platform follows the principle of least privilege:

> An identity should receive only the permissions required to perform its intended function.

For example, an infrastructure deployment role should not automatically receive unrestricted access to every AWS service.

---

## 5. GitHub OIDC

GitHub OIDC will be the preferred mechanism for authenticating GitHub Actions to AWS.

The advantages include:

- No permanent AWS access keys stored in GitHub.
- Short-lived AWS credentials.
- IAM-controlled trust relationships.
- Ability to restrict access to specific repositories.
- Ability to restrict access to specific branches or environments.
- Easier credential revocation.
- Reduced impact from credential leakage.

The trust relationship will eventually be restricted using GitHub OIDC claims.

Example conceptual restriction:

```text
Repository:
mr-ffo/cloud-platform-pipeline

Environment:
production

Branch:
main
```

This means an unrelated repository or branch should not automatically be able to assume the production deployment role.

---

## 6. IAM Roles

The platform will use IAM roles rather than embedding AWS credentials into application code or infrastructure configuration.

We expect to eventually have separate roles for different responsibilities.

Example:

```text
GitHub
 |
 +-- Development Deployment Role
 |
 +-- Staging Deployment Role
 |
 +-- Production Deployment Role
 |
 +-- Application Runtime Roles
 |
 +-- Monitoring / Operational Roles
```

Each role will have a clearly defined responsibility.

Production roles will have stronger trust restrictions and permissions than development roles.

---

## 7. Least Privilege

Least privilege will be applied at multiple levels.

### Identity level

A developer or workflow receives only the permissions required for its job.

### Environment level

Development credentials should not automatically provide production access.

### Service level

An application that only needs S3 access should not receive unrestricted access to EC2, IAM, databases, or other unrelated services.

### Resource level

Where practical, permissions will target specific resources rather than entire AWS services.

For example:

```text
Preferred:

s3:GetObject
on:
arn:aws:s3:::specific-bucket/*

Instead of:

s3:*
on:
*
```

Broad permissions may be temporarily required during early infrastructure bootstrapping, but they should be reviewed and reduced before production.

---

## 8. Secret Management

Secrets are sensitive values that should not be committed to source control.

Examples include:

- Database passwords
- API keys
- OAuth client secrets
- Encryption keys
- Third-party service credentials
- Application authentication secrets

These values will not be placed directly inside TypeScript infrastructure code.

---

## 9. AWS Secrets Manager

AWS Secrets Manager will be the primary secret store for application and runtime secrets.

Example:

```text
AWS Secrets Manager

platform/dev/database
platform/staging/database
platform/prod/database

platform/dev/api
platform/staging/api
platform/prod/api
```

Applications will retrieve secrets at runtime when appropriate rather than receiving permanent credentials through source code.

Secrets Manager also provides capabilities for lifecycle management and secret rotation.

---

## 10. GitHub Secrets

GitHub Secrets will be used only for secrets that specifically belong to GitHub workflows.

GitHub Secrets should not become a replacement for a proper application secret-management system.

Most importantly:

```text
DO NOT:

AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

as permanent credentials for normal GitHub-to-AWS deployments.

Instead:

```text
GitHub Actions
      |
      v
OIDC
      |
      v
AWS IAM
      |
      v
Temporary AWS credentials
```

This significantly reduces the risk associated with long-lived AWS credentials.

---

## 11. Pulumi Secrets

Pulumi configuration may contain sensitive infrastructure configuration.

When a value must be stored in Pulumi configuration, it should use Pulumi's encrypted secret mechanism rather than plaintext configuration.

Conceptually:

```text
Pulumi configuration

Normal value:
aws:region

Secret value:
databasePassword
```

Secret values should be encrypted by Pulumi and should never be committed as plaintext.

Pulumi state itself must also be treated as sensitive infrastructure information.

The state backend and access model will therefore be explicitly designed before production deployment.

---

## 12. Environment Separation

The platform will support multiple environments.

```text
Development
     |
     v
Staging
     |
     v
Production
```

Each environment should have independent:

- Configuration
- IAM permissions
- Secrets
- Infrastructure state
- Deployment controls

Development access should not automatically grant production access.

Production deployments should eventually require stronger controls such as protected environments and explicit approval.

---

## 13. Secret Rotation

Secrets should not be considered permanent.

The architecture will support rotation for credentials where the underlying AWS service or application supports it.

A future production implementation may include:

```text
Secret
  |
  v
Rotation mechanism
  |
  v
New credential
  |
  v
Application retrieves updated value
```

Rotation procedures must also account for applications that cache credentials.

---

## 14. Repository Security

The repository will include automated security checks.

Current controls include:

### Gitleaks

Gitleaks scans the repository for accidentally committed secrets.

### npm audit

Dependencies are checked for known vulnerabilities.

### TypeScript compilation

Infrastructure code must compile successfully.

### Prettier

Infrastructure code must follow consistent formatting.

### ESLint

Infrastructure code is checked for common code-quality issues.

Future controls may include:

- Infrastructure-as-code security scanning
- Dependency update automation
- Container image scanning
- Software composition analysis
- AWS configuration checks
- Security-focused CI policies

---

## 15. Production Security Boundaries

Production infrastructure will have stronger security boundaries than development infrastructure.

Conceptually:

```text
                    GitHub
                       |
                OIDC authentication
                       |
             +---------+---------+
             |                   |
             v                   v
          DEV IAM             PROD IAM
             |                   |
             v                   v
          DEV AWS             PROD AWS
             |                   |
             v                   v
        DEV Secrets         PROD Secrets
```

The production IAM role should only trust the GitHub repository and deployment context required for production deployment.

---

## 16. Threat Model

The platform considers the following major threats.

### Compromised GitHub workflow

Risk:

An attacker modifies a workflow and attempts to access AWS.

Mitigations:

- OIDC
- Restricted IAM trust policies
- Least-privilege IAM policies
- Protected branches
- Protected environments
- Code review
- Audit logging

---

### Leaked application secret

Risk:

A database password or API key becomes exposed.

Mitigations:

- AWS Secrets Manager
- Secret rotation
- No plaintext secrets in source control
- Repository secret scanning
- Least-privilege access

---

### Developer accidentally modifies production

Risk:

An infrastructure change is deployed to production unintentionally.

Mitigations:

- Environment separation
- Protected production environment
- Restricted production IAM role
- Pull requests
- Deployment approvals
- Audit logging

---

### Excessive IAM permissions

Risk:

A compromised identity can access unrelated AWS resources.

Mitigations:

- Least privilege
- Resource-specific permissions
- Separate roles
- IAM policy review
- Regular permission reduction

---

### Infrastructure state exposure

Risk:

Pulumi state contains sensitive infrastructure metadata or configuration.

Mitigations:

- Secure state backend
- Access control
- Encryption
- Restricted CI/CD access
- Appropriate backup and recovery controls

---

## 17. Security Principles

This project follows these principles:

### Principle 1 — No long-lived cloud credentials

Use identity federation and temporary credentials whenever possible.

### Principle 2 — Least privilege

Grant only the permissions required.

### Principle 3 — Assume compromise

Design the platform so that compromise of one component does not automatically compromise the entire platform.

### Principle 4 — Defense in depth

Use multiple independent security controls.

```text
GitHub security
      +
OIDC
      +
IAM
      +
Secrets Manager
      +
Network security
      +
Encryption
      +
Monitoring
      +
Audit logging
```

### Principle 5 — Security as code

Security controls should be reproducible and reviewable through infrastructure-as-code.

### Principle 6 — Environment isolation

Development, staging, and production should have clear security boundaries.

---

## 18. Future Security Work

The following security capabilities will be implemented in later milestones:

- GitHub OIDC provider
- IAM deployment roles
- IAM least-privilege policies
- AWS Secrets Manager
- S3 security hardening
- KMS encryption
- VPC security
- Security groups
- CloudTrail
- CloudWatch security monitoring
- Infrastructure security scanning
- Production environment protection
- Backup and disaster recovery
- Security incident response procedures
- Cost and resource governance

---

## 19. Architecture Decision

The platform adopts the following security model:

```text
GitHub
   |
   | OIDC
   v
AWS IAM
   |
   | Temporary credentials
   v
AWS Infrastructure
   |
   +--------------------+
   |                    |
   v                    v
AWS Services      Secrets Manager
```

GitHub Secrets will remain available for GitHub-specific secrets.

AWS Secrets Manager will be used for application/runtime secrets.

Pulumi encrypted secrets will be used where sensitive infrastructure configuration needs to exist in Pulumi configuration.

Long-lived AWS access keys will not be used as the normal GitHub-to-AWS authentication mechanism.

This architecture establishes the security foundation for the remaining platform implementation.
