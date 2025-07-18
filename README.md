## Overview
This project provisions a secure, Free Tier-compliant AWS environment featuring:
- **VPC** with public/private subnets
- **Bastion host** (SSH access only)
- **Private EC2** running Dockerized NGINX
- **S3 bucket** for logs (versioned, private)
- **IAM roles** with least privilege

## Architecture Diagram
```plaintext
                            +---------------------+
                            |   Your Home IP      |
                            |    (x.x.x.x)        |
                            +----------+----------+
                                       | SSH (Port 22)
                                       v
+---------------------------------------------------------------------------+
|                           AWS Cloud                                       |
|                                                                           |
|  +---------------------+       +---------------------+                    |
|  |    Public Subnet    |       |    Private Subnet   |                    |
|  | +-----------------+ |       | +-----------------+ |                    |
|  | |   Bastion Host  | | SSH   | |  App EC2        | |                    |
|  | |  (t2.micro)     +---------+>|  (t2.micro)     | |                    |
|  | |  SG: Allow SSH  | |       | |  - Docker       | |  +---------------+ |
|  | |  from Home IP   | |       | |  - NGINX:80     | |  |  S3 Bucket    | |
|  | +--------+--------+ |       | +--------+--------+ |  | (Log Storage) | |
|  |          |          |       |          |          |  +-------+-------+ |
|  +----------+----------+       +----------+----------+          |         |
|            |                                                    |         |
|            v                                                    |         |
|  +---------+-----------+                                        |         |
|  | Internet Gateway    |                                        |         |
|  | (Public Route)      |                                        |         |
|  +---------------------+                                        |         |
|                                                                 |         |
|  +-------------------------------------------------------------+          |
|  | VPC (10.0.0.0/16)                                                 |    |
|  +-------------------------------------------------------------------+    |
+---------------------------------------------------------------------------+
```
## Module
|Module|Description|
|------|-----------|
|**template.yaml** | Contains the cloudformation template. |
|**parameters.json** | Contains parameters of cloudformation template |
|**devops-kp.pem** | Key pair for SSH |
|**validation.txt** | Contains validations |

## Deploy instructions:
```
aws cloudformation deploy \
  --template-file template.yml \
  --stack-name devops-assignment \
  --parameter-overrides file://parameters.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```
## SSH tunnel instructions to access the app via the bastion
**Via SSH Tunnel**
```
ssh -i devops-kp.pem -L 8080:<PrivateEC2IP>:80 ec2-user@<BastionIP> -N
```
**Manual SSH Jump**
```
# Connect to bastion
ssh -i devops-kp.pem ec2-user@<BastionIP>

# From bastion, connect to private EC2
ssh -i devops-kp.pem ec2-user@<PrivateEC2IP>

# Verify app
curl http://localhost:80
```
## Design and security decisions
**Infrastructure Details**
```plaintext
[Your Computer] → SSH → [Bastion (t2.micro)] → SSH → [App EC2 (t2.micro)]
                                      ↓
                              [S3 (Log Storage)]
```
**Securty Design**
- **Bastion Host:**
  - Only allows SSH from whitelisted IP (LocalIP/32)
  - No persistent storage of private keys
- **Private EC2:**
  - No public IP
  - Security group allows SSH only from bastion
  - IAM role with S3 write permissions (least privilege)
- **S3 Bucket:**
  - Versioning enabled
  - Block all public access
  - Encrypted at rest

## Cleanup
```
aws cloudformation delete-stack --stack-name devops-assignment
aws s3 rb s3://artsalva-applications-logs --force  # Only if empty
```
