# Terraform State Management

## 1. Remote State with Locking

**Problem:** Local state files get corrupted with concurrent applies

**Solution:** Use remote backend with state locking

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

**Create S3 backend with locking:**

```hcl
# backend.tf
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Enable locking with DynamoDB
resource "aws_dynamodb_table" "terraform_locks" {
  name           = "terraform-locks"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

## 2. Backend Configuration

**AWS S3 + DynamoDB (recommended):**

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

**How locking works:**
- Team member runs `terraform apply`
- Terraform creates lock entry in DynamoDB
- Other team members can't apply (blocked)
- Lock released when apply completes
- Corrupted state prevented

**Initialize backend:**

```bash
# First time
terraform init

# Change backend
terraform init -backend-config="bucket=new-bucket"

# Verify backend
terraform backend show
```

## 3. Workspaces for Multiple Environments

**Isolate state per environment:**

```hcl
# Workspaces use same S3 bucket, different keys:
# dev:      s3://bucket/dev/terraform.tfstate
# staging:  s3://bucket/staging/terraform.tfstate
# prod:     s3://bucket/prod/terraform.tfstate

variable "environment" {
  default = terraform.workspace
}

terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "${terraform.workspace}/terraform.tfstate"  # Cannot interpolate!
    region = "us-east-1"
  }
}
```

**Better approach - use backend config:**

```bash
# dev-backend.tfvars
bucket = "my-terraform-state"
key    = "dev/terraform.tfstate"
region = "us-east-1"

# Initialize with specific backend
terraform init -backend-config=dev-backend.tfvars

# Switch workspaces
terraform workspace list
terraform workspace new staging
terraform workspace select prod
terraform workspace show
```

## 4. State File Security

**Encrypt state in transit and at rest:**

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true              # Server-side encryption
    dynamodb_table = "terraform-locks"
  }
}
```

**IAM policy for state access:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:role/terraform-role"
      },
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-terraform-state"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:role/terraform-role"
      },
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-terraform-state/*/terraform.tfstate"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:role/terraform-role"
      },
      "Action": ["dynamodb:DescribeTable", "dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:DeleteItem"],
      "Resource": "arn:aws:dynamodb:*:ACCOUNT:table/terraform-locks"
    }
  ]
}
```

## 5. State File Management Commands

**Inspect state:**

```bash
# List all resources in state
terraform state list

# Show specific resource
terraform state show aws_instance.web

# Show full state
terraform state pull > backup.tfstate
```

**Modify state (dangerous!):**

```bash
# Remove resource from state (keep in AWS)
terraform state rm aws_instance.web

# Move resource to different state
terraform state mv aws_instance.old aws_instance.new

# Replace resource (force recreate)
terraform taint aws_instance.web
terraform apply  # Destroys and recreates

# Untaint (don't recreate)
terraform untaint aws_instance.web
```

## 6. State Backup and Recovery

**Automatic versioning with S3:**

```hcl
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

**Manual backup:**

```bash
# Daily backup
terraform state pull > backups/state-$(date +%Y%m%d).json

# Automated with cron:
# 0 2 * * * cd /path/to/terraform && terraform state pull > backups/state-$(date +\%Y\%m\%d).json
```

**Recovery from backup:**

```bash
# Restore state file
terraform state push backups/state-20240101.json

# Verify before applying
terraform plan
```

## 7. Lock Management

**View active locks:**

```bash
# AWS CLI
aws dynamodb scan \
  --table-name terraform-locks \
  --region us-east-1

# Shows:
# {
#   "LockID": "my-terraform-state/prod/terraform.tfstate",
#   "Digest": "...",
#   "Operation": "OperationTypeApply",
#   "Info": "...",
#   "Who": "user@hostname",
#   "Version": "1.0.0",
#   "Created": "2024-01-15T10:30:00Z"
# }
```

**Force unlock (last resort):**

```bash
# If lock is stuck from crash
terraform force-unlock <LOCK_ID>

# Or manually via AWS
aws dynamodb delete-item \
  --table-name terraform-locks \
  --key '{"LockID":{"S":"my-terraform-state/prod/terraform.tfstate"}}' \
  --region us-east-1
```

## 8. State File Checklist

- [ ] Remote backend configured (S3, Terraform Cloud, etc.)
- [ ] State locking enabled (DynamoDB for S3)
- [ ] Encryption at rest enabled
- [ ] Encryption in transit enabled
- [ ] S3 versioning enabled for recovery
- [ ] IAM access restricted to team members
- [ ] Regular backups automated
- [ ] State file never in version control
- [ ] Sensitive values marked as sensitive
- [ ] .gitignore includes terraform files

