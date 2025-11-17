# Module 1.2: AWS Core Services - Notes

## Project Completion Status:  COMPLETED

**Date Completed**: 2025-11-17

---

## What I Accomplished

Successfully set up AWS environment and pushed Docker image to ECR:
-  AWS account configured with IAM user
-  AWS CLI installed and configured locally
-  ECR repository created (`docker-api`)
-  Docker image pushed to ECR
-  CloudWatch log group created
-  Explored default VPC and subnets
-  Billing alerts configured

---

## Key Learnings

### 1. IAM (Identity and Access Management)
- **Never use root account** for daily work - created IAM user instead
- **IAM Policies**: Control what actions are allowed (JSON format)
- **Two types of roles for ECS**:
  - Task Execution Role: ECS pulls images, writes logs
  - Task Role: Application accesses AWS resources (S3, DynamoDB, etc.)
- **Access Keys**: Used for AWS CLI authentication (stored in `~/.aws/credentials`)

### 2. ECR (Elastic Container Registry)
- **Private Docker registry** integrated with AWS
- **Repository must exist before pushing** (unlike Docker Hub)
- **Authentication**: `aws ecr get-login-password` gets temporary token (12 hours)
- **URI format**: `<account-id>.dkr.ecr.<region>.amazonaws.com/<repo-name>:<tag>`
- **Image scanning**: Can enable vulnerability scanning on push
- **Lifecycle policies**: Auto-delete old images to save costs

### 3. VPC (Virtual Private Cloud)
- **Default VPC** created automatically in each region
- **CIDR blocks**: Define IP ranges (e.g., `172.31.0.0/16`)
- **Subnets**: Subdivisions across availability zones
- **For ECS**: Tasks run in VPC subnets with security groups
- **Public vs Private subnets**: Public has internet gateway, private uses NAT

### 4. CloudWatch
- **Log Groups**: One per service (e.g., `/ecs/docker-api`)
- **Log Streams**: One per container instance
- **Retention policies**: Set to save costs (7, 30, 90 days)
- **Metrics**: CPU, memory, network for ECS tasks
- **Free tier**: 5 GB ingestion/month

### 5. AWS CLI
- **Configuration**: `aws configure` sets up credentials
- **Credentials location**: `~/.aws/credentials` and `~/.aws/config`
- **Region matters**: Must specify for ECR operations
- **Verify setup**: `aws sts get-caller-identity`

---

## Challenges & Solutions

### Challenge 1: Repository Name Typo
**Problem**: Tagged image as `docker-apiatest` instead of `docker-api`
**Error**: "The repository with name 'docker-apiatest' does not exist"
**Solution**: Re-tagged image with correct repository name
**Learning**: Pay attention to repository names when tagging for ECR

### Challenge 2: Understanding ECR Authentication
**Problem**: Confused about where Docker credentials are stored
**Solution**: Learned that `docker login` stores credentials locally in `~/.docker/config.json`
**Learning**: ECR tokens expire after 12 hours, need to re-authenticate

---

## Commands I Mastered

```bash
# AWS CLI Setup
aws configure
aws sts get-caller-identity

# ECR Operations
aws ecr create-repository --repository-name docker-api --region us-east-1
aws ecr describe-repositories --repository-names docker-api
aws ecr list-images --repository-name docker-api

# ECR Authentication
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  988899761172.dkr.ecr.us-east-1.amazonaws.com

# Docker + ECR
docker tag docker-api:latest 988899761172.dkr.ecr.us-east-1.amazonaws.com/docker-api:latest
docker push 988899761172.dkr.ecr.us-east-1.amazonaws.com/docker-api:latest
docker pull 988899761172.dkr.ecr.us-east-1.amazonaws.com/docker-api:latest

# VPC Exploration
aws ec2 describe-vpcs --query 'Vpcs[?IsDefault==`true`]'
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-xxx"

# CloudWatch Logs
aws logs create-log-group --log-group-name /ecs/docker-api
aws logs put-retention-policy --log-group-name /ecs/docker-api --retention-in-days 7
aws logs tail /ecs/docker-api --follow
```

---

## My AWS Resources

**Account ID**: `988899761172`
**Region**: `us-east-1`
**ECR Repository**: `docker-api`
**ECR URI**: `988899761172.dkr.ecr.us-east-1.amazonaws.com/docker-api`
**VPC**: Default VPC
**Log Group**: `/ecs/docker-api`

---

## Security Best Practices Learned

-  Enable MFA on root account
-  Use IAM user for daily work (not root)
-  Set up billing alerts to avoid unexpected charges
-  Use principle of least privilege for IAM policies
-  Never commit AWS access keys to git
-  ECR image scanning enabled for vulnerabilities
-  Set CloudWatch log retention to control costs

---

## Cost Awareness

**Free Tier Usage**:
- ECR: 500 MB storage/month (using ~190 MB for 1 image)
- CloudWatch Logs: 5 GB/month (minimal usage)
- VPC: Free to use
- IAM: Free to use

**Estimated Monthly Cost**: $0 (within free tier)

---

## How Services Connect for ECS

```
Developer ’ ECR (stores images)
            “
       ECS Task Definition (references ECR image)
            “
       ECS Service (runs in VPC)
            “
       Running Containers (logs to CloudWatch)
```

---

## What's Next

1.  Module 1.2 Complete - Ready for Module 1.3
2. í Module 1.3: Terraform Fundamentals
3. =. Phase 2: Deploy Docker image from ECR to ECS
4. =. Later: Automate with CI/CD

---

## Resources Used

- [AWS Free Tier](https://aws.amazon.com/free/)
- [ECR User Guide](https://docs.aws.amazon.com/AmazonECR/latest/userguide/)
- [AWS CLI Documentation](https://docs.aws.amazon.com/cli/)
- Module 1.2 Lecture & Project Guide

---

**Status**:  Ready to move to Module 1.3 (Terraform)
