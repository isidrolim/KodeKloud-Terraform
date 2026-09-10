# KodeKloud Terraform - Task 001: Create Key Pair Using Terraform

## Scenario

The Nautilus DevOps team is beginning its AWS infrastructure migration using Terraform.

The first task was to create an RSA key pair using Terraform, register its public key with AWS, and securely save the corresponding private key on the Terraform host.

## Requirement

- Create an AWS key pair named `datacenter-kp`.
- Key type must be **RSA**.
- Save the private key as:
  `/home/bob/datacenter-kp.pem`
- Use `/home/bob/terraform/main.tf`.
- Do not replace the existing `provider.tf`.

## Initial State

The Terraform working directory already contained:

```text
/home/bob/terraform/
├── provider.tf
└── README.MD
```

The AWS provider configuration was already provided by the lab.

## Concept

The key pair consists of a public and private key.

```text
tls_private_key
      │
      ├── Public Key ──► aws_key_pair ──► AWS
      │                                  datacenter-kp
      │
      └── Private Key ─► local_file ────► /home/bob/datacenter-kp.pem
```

Three Terraform resources were required:

- `tls_private_key` — generates the RSA key pair.
- `aws_key_pair` — registers the generated public key with AWS.
- `local_file` — writes the generated private key to the local filesystem.

The private key remains on the Terraform host and is not uploaded to AWS.

## Create `main.tf`

```bash
cd /home/bob/terraform
touch main.tf
```

Added the following configuration:

```hcl
resource "tls_private_key" "datacenter_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "datacenter_kp" {
  key_name   = "datacenter-kp"
  public_key = tls_private_key.datacenter_key.public_key_openssh
}

resource "local_file" "private_key" {
  content         = tls_private_key.datacenter_key.private_key_pem
  filename        = "/home/bob/datacenter-kp.pem"
  file_permission = "0600"
}
```

## Initialize Terraform

Initialize the working directory and required providers:

```bash
terraform init
```

Initially, the AWS and TLS providers were available.

After adding the `local_file` resource, `terraform validate` reported:

```text
Error: Missing required provider

This configuration requires provider
registry.terraform.io/hashicorp/local,
but that provider isn't available.
```

The `local_file` resource introduced a dependency on the HashiCorp Local provider.

Reinitialize Terraform:

```bash
terraform init
```

Terraform detected and installed the missing `hashicorp/local` provider.

## Format and Validate

Format the Terraform configuration:

```bash
terraform fmt
```

Validate it:

```bash
terraform validate
```

Validation completed successfully.

## Review the Execution Plan

Before making any infrastructure changes:

```bash
terraform plan
```

Terraform reported:

```text
Plan: 3 to add, 0 to change, 0 to destroy.
```

The plan showed that Terraform would create:

1. RSA private/public key material.
2. AWS EC2 key pair.
3. Local private key file.

At this point the resources did **not** exist yet. `terraform plan` only previews the proposed changes.

## Apply

Create the resources:

```bash
terraform apply
```

Review the proposed changes and enter:

```text
yes
```

Terraform then created all three resources.

## Validation

### Verify the Private Key

```bash
ls -lah /home/bob/
```

Confirmed:

```text
-rw------- ... datacenter-kp.pem
```

This verifies:

- `/home/bob/datacenter-kp.pem` exists.
- Permissions are `0600`.
- Only the owner has read/write permissions.

### Verify the AWS Key Pair

```bash
aws ec2 describe-key-pairs
```

The AWS response confirmed:

```text
"KeyName": "datacenter-kp"
```

Therefore both sides of the dependency path were successfully created:

```text
Terraform
   │
   ├── AWS EC2 ──► datacenter-kp       ✓
   │
   └── Linux ────► datacenter-kp.pem   ✓
```

## Lessons Learned

- `terraform init` initializes the working directory and installs required providers.
- Adding a resource that requires a new provider may require running `terraform init` again.
- `terraform fmt` standardizes Terraform configuration formatting.
- `terraform validate` verifies that the configuration is valid.
- `terraform plan` previews changes but does not create resources.
- `terraform apply` performs the proposed infrastructure changes.
- Terraform automatically understands dependencies when one resource references attributes from another resource.
- AWS stores the public key while the private key must remain protected.
- Private SSH keys should use restrictive permissions such as `0600`.

## Engineering Insight

A safe Terraform workflow is:

```text
Write Configuration
        ↓
terraform init
        ↓
terraform fmt
        ↓
terraform validate
        ↓
terraform plan
        ↓
Review Proposed Changes
        ↓
terraform apply
        ↓
Validate Actual Resources
```

A successful `terraform apply` is not the end of validation. Always verify the original success criteria against the actual infrastructure and system state.

## Knowledge Check

1. What is the difference between `terraform plan` and `terraform apply`?
2. Why did adding `local_file` require running `terraform init` again?
3. Why is only the public key registered with AWS?
4. Why should the private key have `0600` permissions?
5. How does Terraform determine that `aws_key_pair` depends on `tls_private_key`?
6. What role does each resource play: `tls_private_key`, `aws_key_pair`, and `local_file`?
