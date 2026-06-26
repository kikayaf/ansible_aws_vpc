# Setup Guide — ansible_aws_vpc

This guide walks you through installing every dependency, configuring your environment, linting the playbooks, and running the full provisioning workflow.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Clone & Virtual Environment](#2-clone--virtual-environment)
3. [Install Python Dependencies](#3-install-python-dependencies)
4. [Install Ansible Collections](#4-install-ansible-collections)
5. [Configure AWS Credentials](#5-configure-aws-credentials)
6. [Update Project Variables](#6-update-project-variables)
7. [Lint & Validate Playbooks](#7-lint--validate-playbooks)
8. [Run the Playbooks](#8-run-the-playbooks)
9. [Teardown / Cleanup](#9-teardown--cleanup)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Prerequisites

| Tool | Minimum version | Check |
|---|---|---|
| **Python** | 3.8+ | `python3 --version` |
| **pip** | latest | `pip --version` |
| **Git** | any | `git --version` |
| **AWS CLI** | v2 (optional but recommended) | `aws --version` |

> **AWS CLI** is optional for running playbooks but very helpful for finding current AMI IDs and verifying created resources.

---

## 2. Clone & Virtual Environment

```bash
git clone https://github.com/kikayaf/ansible_aws_vpc.git
cd ansible_aws_vpc

# Create and activate an isolated Python virtual environment (recommended)
python3 -m venv .venv
source .venv/bin/activate        # Linux / macOS
# .venv\Scripts\activate.bat     # Windows (cmd)
# .venv\Scripts\Activate.ps1     # Windows (PowerShell)
```

Verify the venv is active — your prompt should show `(.venv)`.

---

## 3. Install Python Dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

This installs:

| Package | Purpose |
|---|---|
| `ansible>=2.10` | Core Ansible engine |
| `boto3` / `botocore` | AWS SDK used by all `ec2_*` modules |
| `ansible-lint` | Playbook linter (see [section 7](#7-lint--validate-playbooks)) |

Verify:

```bash
ansible --version          # should show 2.10.x or later
python3 -c "import boto3; print(boto3.__version__)"
ansible-lint --version
```

---

## 4. Install Ansible Collections

```bash
ansible-galaxy collection install -r requirements.yml
```

This installs `amazon.aws` and `community.aws` into `./collections/` (the path configured in `ansible.cfg`).

Verify:

```bash
ansible-galaxy collection list | grep -E "amazon\.aws|community\.aws"
```

Expected output (versions may be higher):

```
amazon.aws   2.x.x
community.aws 2.x.x
```

> **To upgrade** already-installed collections to their latest compatible version:
> ```bash
> ansible-galaxy collection install -r requirements.yml --upgrade
> ```

---

## 5. Configure AWS Credentials

Both playbooks connect to **`us-east-2` (Ohio)**. Your IAM user or role needs permissions to create and manage VPC and EC2 resources.

### Option A — AWS credentials file (recommended)

```bash
aws configure
# AWS Access Key ID:     <your-key-id>
# AWS Secret Access Key: <your-secret>
# Default region name:   us-east-2
# Default output format: json
```

### Option B — Environment variables

```bash
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_DEFAULT_REGION="us-east-2"
```

### Verify credentials work

```bash
aws sts get-caller-identity
# Should return your Account ID and ARN — if this fails, fix credentials first.
```

### Minimum IAM permissions

```
ec2:CreateVpc, DescribeVpcs, DeleteVpc
ec2:CreateSubnet, DescribeSubnets, DeleteSubnet
ec2:CreateInternetGateway, AttachInternetGateway, DetachInternetGateway, DeleteInternetGateway
ec2:CreateNatGateway, DescribeNatGateways, DeleteNatGateway
ec2:AllocateAddress, DescribeAddresses, ReleaseAddress
ec2:CreateRouteTable, CreateRoute, AssociateRouteTable, DeleteRouteTable
ec2:CreateKeyPair, DescribeKeyPairs, DeleteKeyPair
ec2:CreateSecurityGroup, AuthorizeSecurityGroupIngress, DeleteSecurityGroup
ec2:RunInstances, DescribeInstances, TerminateInstances
ec2:CreateTags
```

---

## 6. Update Project Variables

### `vars/vpc_setup`

Review the CIDR ranges and region.  The defaults are safe for a fresh AWS account but adjust if they overlap with existing VPCs.

```yaml
vpc_name:    "profile-vpc"
vpcCIDR:     '172.20.0.0/16'
region:      "us-east-2"
state:       present      # change to 'absent' to tear everything down
```

### `vars/bastion_host` ← **must update before running**

```yaml
bastion_ami: ami-022661f8a4a1b91cf   # Verify this AMI still exists in us-east-2
region:      us-east-2
MYIP:        YOUR_IP/32              # ← replace with your actual public IP
```

Find your current public IP:

```bash
curl -s https://checkip.amazonaws.com
# Replace MYIP with: <output>/32
```

Find the latest Amazon Linux 2 AMI in us-east-2:

```bash
aws ec2 describe-images \
  --region us-east-2 \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text
```

---

## 7. Lint & Validate Playbooks

Linting is configured in `.ansible-lint`.  Run it before every commit to catch syntax errors, deprecated module usage, and style issues.

### Run ansible-lint

```bash
# Lint all playbooks in the project
ansible-lint

# Lint a specific playbook
ansible-lint vpc_setup.yml
ansible-lint bastion_instance.yml
```

Exit codes:

| Code | Meaning |
|---|---|
| `0` | No violations |
| `1` | Violations found |
| `2` | Fatal error (parse failure, bad config) |

### Syntax check (no AWS calls)

Syntax checking validates YAML structure and Jinja2 templating without connecting to AWS — fast and safe to run any time:

```bash
ansible-playbook --syntax-check vpc_setup.yml
ansible-playbook --syntax-check bastion_instance.yml
```

### Dry run (check mode)

Check mode connects to AWS and reports what *would* change without actually making changes:

```bash
ansible-playbook vpc_setup.yml --check
ansible-playbook bastion_instance.yml --check
```

> **Note:** Some AWS modules do not fully support check mode and will report `skipped`.  Syntax-check is the most reliable pre-flight validation.

### Full pre-commit workflow

```bash
ansible-lint \
  && ansible-playbook --syntax-check vpc_setup.yml \
  && ansible-playbook --syntax-check bastion_instance.yml \
  && echo "All checks passed ✓"
```

---

## 8. Run the Playbooks

> **Important:** Always run `vpc_setup.yml` first.  It writes `vars/output_vars` with VPC/subnet IDs that `bastion_instance.yml` reads.

### Step 1 — Provision VPC and networking

```bash
ansible-playbook vpc_setup.yml
```

What happens:
1. Creates the VPC (`172.20.0.0/16`)
2. Creates 3 public + 3 private subnets across `us-east-2a/b/c`
3. Attaches an Internet Gateway
4. Creates a NAT Gateway in `pubsub1`
5. Creates public and private route tables
6. Writes all resource IDs to `vars/output_vars`

Expected run time: **3–6 minutes** (NAT Gateway provisioning is the slowest step).

### Step 2 — Provision Bastion Host

```bash
ansible-playbook bastion_instance.yml
```

What happens:
1. Reads `vars/output_vars` (produced by Step 1)
2. Creates an EC2 key pair (`profile-key`) and saves `bastion-key.pem` locally
3. Creates security group allowing SSH only from `MYIP`
4. Launches a `t2.micro` EC2 instance in `pubsub1`

Expected run time: **1–2 minutes**.

### Useful flags

```bash
# Increase verbosity for debugging
ansible-playbook vpc_setup.yml -v       # basic
ansible-playbook vpc_setup.yml -vvv     # full module output

# Run only specific tags (once tags are added)
ansible-playbook vpc_setup.yml --tags subnets

# Limit to a single task by name
ansible-playbook vpc_setup.yml --start-at-task "Create vprofile VPC"
```

---

## 9. Teardown / Cleanup

To destroy all provisioned resources and avoid ongoing AWS charges, change `state` to `absent` in `vars/vpc_setup`:

```yaml
state: absent
```

Then re-run both playbooks in **reverse order**:

```bash
# 1. Remove Bastion Host, security group, and key pair first
ansible-playbook bastion_instance.yml

# 2. Remove NAT Gateway (this can take several minutes), subnets, route tables,
#    Internet Gateway, and finally the VPC itself
ansible-playbook vpc_setup.yml
```

> **Important:** The NAT Gateway must be fully deleted before the VPC can be removed.  If the playbook fails on VPC deletion, wait a couple of minutes and re-run `vpc_setup.yml`.

---

## 10. Troubleshooting

### `boto3` / `botocore` not found

```
ModuleNotFoundError: No module named 'boto3'
```

Make sure your virtual environment is activated before running any `ansible-playbook` command:

```bash
source .venv/bin/activate
```

---

### `[WARNING]: provided hosts list is empty`

The playbooks use `hosts: localhost` but Ansible can't find an inventory.  This is fixed by `ansible.cfg` pointing to `./inventory`.  If you see this warning, verify `ansible.cfg` is present:

```bash
ansible --version | grep "config file"
# Should show: config file = /path/to/ansible_aws_vpc/ansible.cfg
```

---

### `Unable to locate credentials`

```
botocore.exceptions.NoCredentialsError: Unable to locate credentials
```

Set up credentials via `aws configure` or export `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` environment variables.

---

### `InvalidAMIID.NotFound`

The `bastion_ami` in `vars/bastion_host` is no longer available in `us-east-2`. Run the AMI lookup command from [section 6](#6-update-project-variables) and update the value.

---

### `VpcLimitExceeded`

AWS accounts are limited to 5 VPCs per region by default.  Delete an unused VPC in `us-east-2` or request a limit increase via the AWS Service Quotas console.

---

### `ConnectionTimeout` during NAT Gateway creation

NAT Gateway provisioning can take 2–5 minutes.  The `wait: yes` parameter in `vpc_setup.yml` handles this automatically.  If the task times out, re-run the playbook — it is idempotent and will pick up where it left off.

---

### ansible-lint reports `fqcn` warnings

The playbooks use short module names (e.g. `ec2_vpc_net` instead of `community.aws.ec2_vpc_net`).  These are suppressed in `.ansible-lint` via `skip_list`.  If you upgrade to ansible-lint 7+ or change the profile to `production`, you will need to add FQCN prefixes to every task.
