# Cloud Infrastructure & Security Monitoring

An AWS lab I built to demonstrate private infrastructure, least-privilege access and a working security alert. The lab ran in `eu-west-2` on 29 September 2026 and was destroyed after validation.

## Architecture

```mermaid
flowchart TB
    TF["Terraform"]

    subgraph FOUNDATION["Secure AWS foundation"]
        direction LR
        VPC["Private VPC + EC2"]
        S3["Encrypted, private S3"]
        IAM["Scoped IAM role"]
    end

    subgraph DETECTION["Detection and response"]
        direction LR
        TEST["Denied API event"] --> TRAIL["CloudTrail"]
        TRAIL --> WATCH["CloudWatch alarm"]
        WATCH --> SNS["SNS email alert"]
    end

    TF --> VPC
    TF --> S3
    TF --> IAM
    IAM --> TEST
    CHECK["Python validation: 7/7 passed"] -.-> VPC
    CHECK -.-> S3
    CHECK -.-> TRAIL

    classDef source fill:#ede9fe,stroke:#7c3aed,color:#2e1065,stroke-width:2px
    classDef secure fill:#e0f2fe,stroke:#0284c7,color:#082f49,stroke-width:2px
    classDef monitor fill:#fef3c7,stroke:#d97706,color:#451a03,stroke-width:2px
    classDef result fill:#dcfce7,stroke:#16a34a,color:#052e16,stroke-width:2px
    class TF source
    class VPC,S3,IAM secure
    class TEST,TRAIL,WATCH monitor
    class SNS,CHECK result
```

Terraform defined the infrastructure and alert path. A Python script ran seven read-only checks against the deployed configuration.

## My approach

I wanted to show that a security control could be tested end to end: a restricted identity makes an API request, AWS records the denial, and an alert reaches a person. I kept the EC2 instance private, limited the S3 reader role to one bucket, and used a dry-run request so the test could not delete the VPC.

The first validation run exposed a credential-provider dependency issue. After resolving it, all seven checks passed. This was a useful reminder to test the validation tooling as carefully as the infrastructure.

## What I verified

| Area | Observed result |
| --- | --- |
| Deployment | Terraform created 25 resources, including a private EC2 instance. |
| Security settings | S3 public access was blocked, default encryption and versioning were enabled, and the instance security group had no inbound or outbound rules. |
| Automated checks | All 7 Python checks passed. |
| Detection | A restricted role's denied dry-run API request appeared in CloudTrail; CloudWatch entered ALARM and SNS delivered an [email notification](alert-email.png). |
| Cleanup | Terraform reported 25 resources destroyed. |

The email screenshot has account information covered. I keep the Terraform and Python source, detailed run notes and full console evidence privately for interview discussion.

This was a disposable test environment. The Python checks run on demand; they are not a continuous compliance service.

**Next iteration:** run the checks on a schedule and measure alert delivery across several tests, rather than relying on a single observed run.
