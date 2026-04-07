# Common State Management Mistakes

## 1. Local State in Version Control

**Pitfall:** Committing terraform.tfstate to Git

```bash
# BAD: Checking in state file
git add terraform.tfstate
git push origin main
# Now state is in GitHub history, visible to all

# GOOD: Add to .gitignore
echo "terraform.tfstate*" >> .gitignore
echo "*.tfvars" >> .gitignore
echo ".terraform/" >> .gitignore
git add .gitignore
```

**Why dangerous:**
- Exposes secrets (passwords, API keys) in state
- Team members see each other's apply operations
- State history visible in Git logs forever
- Impossible to revoke access to past secrets

## 2. No State Locking

**Pitfall:** Two team members applying simultaneously

```
Time  Person A              Person B
----  --------              --------
1:00  terraform plan
1:02  terraform apply (starts)
1:03                        terraform plan
1:04  terraform apply (gets resources)
1:05                        terraform apply (overwrites!)
      State now corrupt!
```

**Solution:**

```hcl
# Enable DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    dynamodb_table = "terraform-locks"  # Must exist!
  }
}
```

**Without locking:**
- Concurrent applies race condition
- State file corruption
- Resource inconsistency
- Data loss possible

## 3. Sensitive Data in State

**Pitfall:** Plain-text secrets in state file

```hcl
# BAD: Password in state
resource "aws_db_instance" "main" {
  allocated_storage    = 20
  engine               = "mysql"
  instance_class       = "db.t2.micro"
  username             = "admin"
  password             = "MyPassword123!"  # In state file!
  skip_final_snapshot  = true
}

# GOOD: Use Secrets Manager
resource "aws_secretsmanager_secret" "db_password" {
  name = "db-password"
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = var.db_password
}

resource "aws_db_instance" "main" {
  # ... other config
  password = jsondecode(aws_secretsmanager_secret_version.db_password.secret_string)["password"]
}
```

**Or mark as sensitive:**

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

resource "aws_db_instance" "main" {
  password = var.db_password
}

# Now won't print in logs:
# terraform apply
# aws_db_instance.main: Modifying... [id=db-xxx]
# aws_db_instance.main: Modification complete! [id=db-xxx]
# (password not shown)
```

## 4. Forgetting .tfvars Files

**Pitfall:** Variables in Git but values exposed

```bash
# .gitignore (MISSING .tfvars)
*.tfstate
*.tfstate.backup
.terraform/
# MISSING: *.tfvars

# Result: This gets committed
terraform.tfvars
# db_password = "secret123"
# api_key = "xxx-yyy-zzz"
```

**Fix:**

```bash
# Add to .gitignore
echo "*.tfvars" >> .gitignore
echo "!*.tfvars.example" >> .gitignore

# Create example file for team
cat > terraform.tfvars.example <<EOF
db_password = "changeme"
api_key = "changeme"
EOF

git add .gitignore terraform.tfvars.example
git rm --cached terraform.tfvars
```

## 5. State File Not Encrypted

**Pitfall:** S3 bucket without encryption

```hcl
# BAD: No encryption
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    # encrypt = true is missing!
  }
}

# GOOD: Enable encryption
terraform {
  backend "s3" {
    bucket  = "my-terraform-state"
    key     = "prod/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true
    dynamodb_table = "terraform-locks"
  }
}
```

**State file contains:**
- Resource IDs
- Configuration values
- Database passwords
- API keys
- Private IPs

## 6. No Backup Strategy

**Pitfall:** Single point of failure

```bash
# BAD: No backup
# If S3 bucket accidentally deleted = disaster

# GOOD: Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-terraform-state \
  --versioning-configuration Status=Enabled

# BETTER: Enable MFA delete protection
aws s3api put-bucket-versioning \
  --bucket my-terraform-state \
  --versioning-configuration Status=Enabled \
  --mfa "arn:aws:iam::ACCOUNT:mfa/user-mfa-device 123456"
```

**Automated backup:**

```bash
#!/bin/bash
# Daily backup script
BACKUP_DIR="/backups/terraform"
mkdir -p $BACKUP_DIR

for workspace in dev staging prod; do
  terraform workspace select $workspace
  terraform state pull > "$BACKUP_DIR/state-${workspace}-$(date +%Y%m%d).json"
done

# Archive older than 30 days
find $BACKUP_DIR -name "*.json" -mtime +30 -exec gzip {} \;
```

## 7. Incorrect Backend Config

**Pitfall:** Backend key paths that collide

```hcl
# BAD: Same key for all environments
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "terraform.tfstate"  # All envs use same file!
  }
}

# GOOD: Unique key per environment
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"  # Unique per env
  }
}

# BETTER: Use workspaces
terraform workspace new prod
terraform workspace new staging
# Terraform creates prod/terraform.tfstate and staging/terraform.tfstate
```

## 8. Emergency Responses

**Scenario: State file corrupted**

```bash
# Step 1: Pull backup
terraform state pull > current-broken.tfstate
terraform state push backups/state-2024-01-14.json

# Step 2: Verify structure
terraform state list
terraform plan

# Step 3: If still issues, rebuild from source
terraform destroy  # Removes all resources
terraform apply    # Recreates from code
```

**Scenario: Lock stuck (apply crashed)**

```bash
# Check lock status
aws dynamodb scan --table-name terraform-locks

# Force unlock
terraform force-unlock <LOCK_ID>

# Verify no partial apply occurred
terraform refresh  # Sync state with real resources
terraform plan
```

**Scenario: Accidental secret in state**

```bash
# 1. Rotate the secret
# 2. Remove from state
terraform state rm aws_db_instance.main
terraform import aws_db_instance.main <resource-id>

# 3. Restore with new secret
terraform apply

# 4. Force state backup with old version deleted
# S3 versioning prevents recovery of old state
```

## Prevention Checklist

- [ ] .gitignore includes *.tfstate, *.tfvars
- [ ] Remote backend configured
- [ ] State locking enabled (DynamoDB)
- [ ] Encryption at rest enabled
- [ ] S3 versioning enabled
- [ ] IAM permissions restrictive
- [ ] Sensitive variables marked
- [ ] Backup strategy documented
- [ ] MFA delete protection on S3
- [ ] Team access control in place

