# AWS Cross-Region Windows EC2 File Replication & Disaster Recovery

## Overview

I built a cross-region file replication and recovery architecture using
Windows EC2, Amazon S3, S3 Cross-Region Replication (CRR), IAM, AWS CLI,
and AWS Systems Manager.

The project uses:

-   Mumbai: `ap-south-1`
-   Hyderabad: `ap-south-2`

The goal is to replicate files from a Windows EC2 instance in Mumbai to
a separate S3 bucket in Hyderabad and recover those files on a Windows
EC2 instance in Hyderabad.

------------------------------------------------------------------------

## Architecture

``` text
Mumbai Region                         Hyderabad Region

Windows EC2                           Windows EC2
C:\TEST                               C:\TEST
    |                                     ^
    | AWS CLI                              | AWS CLI
    v                                     |
mum-main-bkt  ───── S3 CRR ─────>  hyd-bkt-2
ap-south-1                         ap-south-2
```

### Data flow

``` text
Mumbai EC2
    ↓
Mumbai S3
    ↓ S3 Cross-Region Replication
Hyderabad S3
    ↓
Hyderabad EC2
```

No VPC peering, VPN, or Transit Gateway is required for the file
replication path.

------------------------------------------------------------------------

# 1. Infrastructure

## Mumbai

-   Custom VPC: `10.10.0.0/16`
-   Subnet
-   Route table
-   Internet Gateway
-   Windows EC2
-   Security Group with RDP restricted to my IP

## Hyderabad

-   Custom VPC: `10.20.0.0/16`
-   Subnet
-   Route table
-   Internet Gateway
-   Windows EC2
-   Security Group with RDP restricted to my IP

Separate CIDR ranges were used to avoid overlapping networks.

------------------------------------------------------------------------

# 2. IAM

I used IAM roles on the EC2 instances instead of storing long-lived AWS
access keys.

### Mumbai

Role:

``` text
Mum-Iam
```

Verified with:

``` powershell
aws sts get-caller-identity
```

Result confirmed:

``` text
arn:aws:sts::742563278558:assumed-role/Mum-Iam/i-0e1f0cec4065cc4cb
```

### Hyderabad

Role:

``` text
hyd-iam-role
```

Verified with:

``` powershell
aws sts get-caller-identity
```

Result confirmed:

``` text
arn:aws:sts::742563278558:assumed-role/hyd-iam-role/i-0eda3e566dc2a343
```

The Hyderabad role was given scoped access to `hyd-bkt-2` and the
`TEST/` prefix.

------------------------------------------------------------------------

# 3. Amazon S3

Two S3 buckets were created:

  Region                     Bucket
  -------------------------- ----------------
  Mumbai (`ap-south-1`)      `mum-main-bkt`
  Hyderabad (`ap-south-2`)   `hyd-bkt-2`

S3 Versioning was enabled on both buckets.

The project uses the `TEST/` prefix:

``` text
mum-main-bkt/TEST/
hyd-bkt-2/TEST/
```

------------------------------------------------------------------------

# 4. AWS CLI and SSM

AWS CLI was installed on both Windows EC2 instances.

Version used during testing:

``` text
aws-cli/2.37.6
```

The instances were verified using:

``` powershell
aws sts get-caller-identity
```

AWS Systems Manager Session Manager was also used to access the Windows
instances without depending entirely on RDP.

------------------------------------------------------------------------

# 5. Mumbai File Upload

The Mumbai EC2 contains:

``` text
C:\TEST
```

The original test file was:

``` text
Rk.txt
```

It was uploaded with:

``` powershell
aws s3 sync C:\TEST s3://mum-main-bkt/TEST
```

Verification:

``` powershell
aws s3 ls s3://mum-main-bkt/TEST/
```

Result:

``` text
2026-09-29 12:39:51         28 Rk.txt
```

------------------------------------------------------------------------

# 6. Cross-Region Replication

I configured S3 Cross-Region Replication:

``` text
mum-main-bkt
     |
     | CRR
     v
hyd-bkt-2
```

The replication scope was:

``` text
TEST/
```

`Rk.txt` existed before the CRR rule was created, so I created a new
test file after the rule was configured.

------------------------------------------------------------------------

# 7. CRR Test

On the Mumbai EC2:

``` powershell
"Hello from Mumbai EC2 - CRR test" | Out-File C:\TEST\crr-test.txt
```

The file was uploaded:

``` powershell
aws s3 sync C:\TEST s3://mum-main-bkt/TEST
```

Output:

``` text
upload: ..\..\TEST\crr-test.txt to s3://mum-main-bkt/TEST/crr-test.txt
```

Mumbai S3 was then verified:

``` powershell
aws s3 ls s3://mum-main-bkt/TEST/
```

Result:

``` text
2026-09-29 12:39:51         28 Rk.txt
2026-09-30 10:00:37         70 crr-test.txt
```

------------------------------------------------------------------------

# 8. Hyderabad CRR Verification

On the Hyderabad EC2:

``` powershell
aws s3 ls s3://hyd-bkt-2/TEST/
```

Result:

``` text
2026-09-30 10:00:37         70 crr-test.txt
```

The object was also verified in the S3 Console.

This confirmed:

``` text
Mumbai S3 → Hyderabad S3
```

through S3 Cross-Region Replication.

------------------------------------------------------------------------

# 9. Hyderabad Recovery Test

I created the destination directory:

``` powershell
New-Item -ItemType Directory -Path C:\TEST -Force
```

Then downloaded the replicated file:

``` powershell
aws s3 sync s3://hyd-bkt-2/TEST C:\TEST
```

Result:

``` text
download: s3://hyd-bkt-2/TEST/crr-test.txt to ..\..\TEST\crr-test.txt
```

The recovered file was verified:

``` powershell
Get-Content C:\TEST\crr-test.txt
```

Output:

``` text
Hello from Mumbai EC2 - CRR test
```

------------------------------------------------------------------------

# 10. End-to-End Result

The complete path was successfully tested:

``` text
Mumbai EC2
    ↓
mum-main-bkt
    ↓ CRR
hyd-bkt-2
    ↓
Hyderabad EC2
    ↓
C:\TEST\crr-test.txt
```

  Test                           Result
  ------------------------------ --------
  Mumbai EC2 → Mumbai S3         PASS
  Mumbai S3 → Hyderabad S3       PASS
  Hyderabad S3 → Hyderabad EC2   PASS
  File content verification      PASS

------------------------------------------------------------------------

# 11. Disaster Recovery Design

The second S3 bucket provides a separate cross-region copy of replicated
data.

If the Mumbai environment becomes unavailable, objects that have already
replicated to `hyd-bkt-2` remain available in the Hyderabad Region.

The Hyderabad EC2 can recover those objects using:

``` powershell
aws s3 sync s3://hyd-bkt-2/TEST C:\TEST
```

### Important limitation

S3 Cross-Region Replication is asynchronous. Data created immediately
before a failure may not have completed replication yet.

Therefore, this architecture provides cross-region replication and
recovery capability, but it is not a complete backup strategy.

------------------------------------------------------------------------
