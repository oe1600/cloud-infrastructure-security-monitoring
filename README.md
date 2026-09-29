# Cloud Infrastructure & Security Monitoring

A secure AWS environment built with EC2, S3 and VPC, using scoped IAM permissions, CloudWatch monitoring and email alerts. Terraform handled the infrastructure, and Python checks validated the network, storage and logging settings.

Completed in `eu-west-2` on 29 September 2026. All resources were removed after testing.

## Architecture

```mermaid
flowchart TB
    TF["Terraform deployment"]
    TF --> VPC
    TF --> S3
    TF --> IAM
    VPC["Private VPC + EC2"]
    S3["Encrypted, private S3"]
    IAM["Scoped IAM role"] --> TEST["Denied API event"]
    TEST --> TRAIL["CloudTrail"]
    TRAIL --> WATCH["CloudWatch alarm"]
    WATCH --> SNS["SNS email alert"]

    classDef source fill:#ede9fe,stroke:#7c3aed,color:#2e1065,stroke-width:2px
    classDef secure fill:#e0f2fe,stroke:#0284c7,color:#082f49,stroke-width:2px
    classDef monitor fill:#fef3c7,stroke:#d97706,color:#451a03,stroke-width:2px
    classDef result fill:#dcfce7,stroke:#16a34a,color:#052e16,stroke-width:2px
    class TF source
    class VPC,S3,IAM secure
    class TEST,TRAIL,WATCH monitor
    class SNS result
```

Terraform defined the infrastructure and alert path. A Python script ran seven read-only checks against the deployed configuration.

## Security design

| Control | Design choice |
| --- | --- |
| Network | The EC2 instance ran in a private subnet with no public IPv4 address. Its security group had no inbound or outbound rules. |
| Storage | The data bucket blocked public access and had default encryption and versioning enabled. |
| Access | A reader role could list the lab bucket and read its objects, without broad S3 or EC2 permissions. |
| Monitoring | CloudTrail sent management events to CloudWatch Logs. A metric filter and alarm watched for denied API activity and notified an SNS email subscription. |

## My approach

I wanted to show that a security control could be tested end to end: a restricted identity makes an API request, AWS records the denial, and an alert reaches a person. I kept the EC2 instance private, limited the S3 reader role to one bucket, and used a dry-run request so the test could not delete the VPC.

The first validation run exposed a credential-provider dependency issue. After resolving it, all seven checks passed. This was a useful reminder to test the validation tooling as carefully as the infrastructure.

## What I verified

| Area | Observed result |
| --- | --- |
| Deployment | Terraform created 25 resources, including a private EC2 instance. |
| Security settings | S3 public access was blocked, default encryption and versioning were enabled, and the instance security group had no inbound or outbound rules. |
| Automated checks | All 7 Python checks passed. |
| Detection | A restricted role's denied dry-run `DeleteVpc` request appeared in CloudTrail. No VPC was deleted. |
| Alert | CloudWatch entered ALARM about four minutes after the recorded denial, and SNS delivered an [email notification](screenshots/alert-email.png). This timing is from one run, not a delivery guarantee. |
| Cleanup | Terraform reported 25 resources destroyed. |

## Evidence from the lab

The screenshots below show the deployed configuration and observed test results. Expand each section to view the evidence.

<details open>
<summary><strong>Private EC2 instance and VPC</strong></summary>

The instance was running in the lab VPC and private subnet, with no public IPv4 address.

![Private EC2 instance and VPC](screenshots/02-vpc-ec2.png)

</details>

<details>
<summary><strong>Security group: no inbound or outbound access</strong></summary>

Both rule lists were empty, matching the isolated network design.

![Security group inbound rules](screenshots/03-security-group-inbound.png)
![Security group outbound rules](screenshots/03-security-group-outbound.png)

</details>

<details>
<summary><strong>S3: public access blocked, encryption and versioning enabled</strong></summary>

The data bucket had Block Public Access enabled, SSE-S3 default encryption and versioning.

![S3 public access protection](screenshots/04-s3-public-access.png)
![S3 default encryption](screenshots/04-s3-encryption.png)
![S3 versioning](screenshots/04-s3-versioning.png)

</details>

<details>
<summary><strong>IAM: bucket-scoped read permissions</strong></summary>

The policy limited list and object-read permissions to the lab data bucket.

![Scoped IAM policy](screenshots/05-iam-policy.png)

</details>

<details>
<summary><strong>CloudTrail: active logging and CloudWatch integration</strong></summary>

CloudTrail was logging, with log-file validation enabled and a CloudWatch Logs destination configured.

![CloudTrail logging configuration](screenshots/06-cloudtrail.png)

</details>

<details>
<summary><strong>Detection: denied event, alarm transition and email</strong></summary>

CloudTrail recorded the denied operation. CloudWatch history shows the transition to ALARM and successful execution of the SNS action, followed by a return to OK.

![Denied API event in CloudTrail](screenshots/08-cloudtrail-denied-event.png)
![CloudWatch alarm history](screenshots/07-alarm-history.png)
![SNS alert email](screenshots/alert-email.png)

</details>

<details>
<summary><strong>Python validation: seven checks passed</strong></summary>

The checks confirmed the S3 settings, empty security-group rules, active CloudTrail logging and CloudWatch Logs integration.

![Seven passing security checks](screenshots/10-validation.png)

</details>

This was a disposable test environment. The Python checks run on demand; they are not a continuous compliance service.

**Next iteration:** run the checks on a schedule and measure alert delivery across several tests, rather than relying on a single observed run.
