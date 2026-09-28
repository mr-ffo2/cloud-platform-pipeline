# Pulumi State Bootstrap and AWS S3 Backend

## Overview

This milestone establishes the persistent state backend for the `cloud-platform-pipeline` infrastructure project.

Pulumi requires state to track the resources it manages. For this project, the state is stored in an AWS S3 bucket rather than Pulumi Cloud.

The state backend is intentionally bootstrapped separately from the main infrastructure stack because the main stack depends on the state backend to operate.

## Architecture

```text
Developer
    |
    v
Pulumi CLI
    |
    v
AWS S3 Pulumi State Backend
    |
    v
Pulumi dev Stack
    |
    v
AWS Infrastructure
```

The state bucket and platform resources have separate responsibilities:

```text
cloud-platform-pipeline-pulumi-state-170248754533
    -> Stores Pulumi state

platform-storage-79c3d86
    -> Platform/application storage
```

## Why S3 Was Selected

AWS S3 was selected as the Pulumi backend because the project is intended to demonstrate practical cloud-platform engineering using AWS.

Using an AWS-managed S3 backend provides hands-on experience with:

- infrastructure state management
- S3 security controls
- encryption
- versioning
- AWS access control
- infrastructure bootstrap
- operational separation of state and application resources

## State Bucket

The Pulumi state bucket is:

```text
cloud-platform-pipeline-pulumi-state-170248754533
```

Region:

```text
us-east-1
```

The bucket was created outside the main Pulumi stack using AWS CLI.

This is intentional.

The main Pulumi stack cannot safely create the bucket that it requires to store its own state without introducing a circular dependency.

## State Bucket Security Controls

The state bucket was configured with the following controls.

### Versioning

S3 versioning is enabled.

Purpose:

- recover previous state versions
- protect against accidental overwrites
- provide an additional recovery mechanism

### Encryption

Server-side encryption uses:

```text
AES256
```

This protects state objects at rest.

A customer-managed KMS key may be introduced later if the platform's security requirements justify the additional key-management complexity.

### Public Access Protection

All four S3 Block Public Access controls are enabled:

```text
BlockPublicAcls=true
IgnorePublicAcls=true
BlockPublicPolicy=true
RestrictPublicBuckets=true
```

Pulumi state must never be publicly accessible.

### Object Ownership

The bucket uses:

```text
BucketOwnerEnforced
```

This disables ACL-based ownership behavior and keeps ownership under the bucket owner.

## Pulumi Backend Configuration

The local Pulumi CLI was configured to use:

```text
s3://cloud-platform-pipeline-pulumi-state-170248754533
```

The backend was verified with:

```bash
pulumi whoami
```

and Pulumi reported the local S3 backend identity.

## Dev Stack

The project uses a `dev` Pulumi stack.

The stack was initialized with:

```bash
pulumi stack init dev
```

The stack configuration includes:

```yaml
aws:region: us-east-1
```

The stack also inherits the project tags defined in `Pulumi.yaml`.

## Secrets Provider

The `dev` stack uses Pulumi's passphrase secrets provider.

The passphrase protects encrypted Pulumi configuration and secrets.

The passphrase must never be committed to Git or stored in repository files.

For future CI/CD implementation, the passphrase will need to be supplied through a secure CI/CD secret mechanism rather than being hard-coded.

## First Infrastructure Deployment

The first infrastructure deployment was intentionally small.

The main infrastructure entry point imports the storage module:

```ts
import "./storage";
```

The storage module creates:

```ts
new aws.s3.Bucket("platform-storage", {
    forceDestroy: false,
});
```

Pulumi preview reported:

```text
2 resources to create
```

The deployment then successfully created the Pulumi stack and the platform storage bucket.

The resulting AWS bucket was:

```text
platform-storage-79c3d86
```

## Verification

The AWS account was verified using:

```bash
aws s3api list-buckets
```

The result confirmed both separate buckets:

```text
cloud-platform-pipeline-pulumi-state-170248754533
platform-storage-79c3d86
```

Pulumi also confirmed the deployed resources:

```text
pulumi:pulumi:Stack
cloud-platform-pipeline-dev

aws:s3/bucket:Bucket
platform-storage

pulumi:providers:aws
default_7_34_0
```

## Engineering Lessons

This milestone demonstrates several important platform-engineering concepts.

### Desired State vs Actual State

Infrastructure code defines desired state.

Pulumi compares desired state against its recorded state and the cloud provider's resources.

```text
Infrastructure Code
        |
        v
Desired State
        |
        v
Pulumi
        |
        v
AWS
        |
        v
Actual Infrastructure
```

### State Is Infrastructure Data

Pulumi state is not ordinary application data.

It represents the resources Pulumi manages and is therefore operationally sensitive.

It must be:

- protected
- encrypted
- recoverable
- access-controlled
- excluded from source control

### Bootstrap Resources Are Different

The state bucket exists to support the infrastructure management system itself.

Therefore it is intentionally separated from:

```text
infra/storage
```

which contains platform resources managed by the main stack.

## Operational Principle

The platform follows this principle:

> Infrastructure required to manage the platform should not depend on the platform it manages.

The Pulumi state backend is therefore bootstrapped independently.

## Future Improvements

The following improvements are planned for later milestones:

- CI/CD authentication through GitHub OIDC
- dedicated AWS IAM roles
- least-privilege permissions
- environment separation
- production state isolation
- improved S3 lifecycle policies
- customer-managed KMS encryption where justified
- state recovery procedures
- monitoring and alerting
- disaster-recovery considerations
- automated infrastructure previews
- controlled production deployments

## Milestone Status

Completed:

- [x] AWS CLI configured
- [x] AWS region configured
- [x] Pulumi CLI available
- [x] Dedicated S3 state bucket created
- [x] State bucket versioning enabled
- [x] State bucket encryption enabled
- [x] Public access blocked
- [x] Bucket ownership enforced
- [x] Pulumi configured to use S3 backend
- [x] `dev` stack initialized
- [x] Pulumi preview completed
- [x] First infrastructure deployment completed
- [x] AWS resource verified

The next milestone is **Storage Security and Hardening**.
