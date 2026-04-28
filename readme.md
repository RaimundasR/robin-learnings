# Lab Task: Deploy Ubuntu 24.04 Droplet with Terraform (DigitalOcean)

## Objective

Provision the smallest Ubuntu 24.04 LTS Droplet on DigitalOcean using Terraform from Windows or WSL2, and connect to it via SSH using a generated key.

---

## Step 0: Create DigitalOcean API Token (Required)

1. Open:
   [https://cloud.digitalocean.com/account/api/tokens](https://cloud.digitalocean.com/account/api/tokens)

2. Click "Generate New Token"

3. Configure:
   Name: terraform-token
   Scope: Read and Write

4. Copy and securely store the token

Note: The token is shown only once.

---

## Step 1: Generate SSH Key

### Option A: Windows Terminal (PowerShell)

```powershell
ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\robin-do-key
```

Files created:

C:\Users<user>.ssh\robin-do-key
C:\Users<user>.ssh\robin-do-key.pub

---

### Option B: WSL2 Ubuntu (Recommended)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/robin-do-key
```

Files created:

/home/<user>/.ssh/robin-do-key
/home/<user>/.ssh/robin-do-key.pub

---

## Step 2: Install Terraform

### Windows

```powershell
winget install Hashicorp.Terraform
```

Verify:

```powershell
terraform version
```

---

### WSL2 Ubuntu

```bash
sudo apt update
sudo apt install -y terraform
```

Verify:

```bash
terraform version
```

---

## Step 3: Create Terraform Project Using Best-Practice Folder Structure

Create the project folder:

```bash
mkdir do-terraform-first-droplet
cd do-terraform-first-droplet
```

Use the following Terraform folder structure:

```text
do-terraform-first-droplet/
├── main.tf
├── provider.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars.example
├── versions.tf
└── README.md
```

File purpose:

| File                     | Purpose                                             |
| ------------------------ | --------------------------------------------------- |
| versions.tf              | Defines Terraform and provider version requirements |
| provider.tf              | Configures the DigitalOcean provider                |
| variables.tf             | Defines input variables                             |
| main.tf                  | Contains infrastructure resources                   |
| outputs.tf               | Prints useful values after deployment               |
| terraform.tfvars.example | Example variable values, safe to share              |
| README.md                | Short project documentation                         |

Do not commit or share sensitive files such as:

```text
terraform.tfvars
.terraform/
.terraform.lock.hcl may be committed, but do not edit it manually
terraform.tfstate
terraform.tfstate.backup
```

---

## Step 3.1: Create versions.tf

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    digitalocean = {
      source  = "digitalocean/digitalocean"
      version = "~> 2.0"
    }
  }
}
```

---

## Step 3.2: Create provider.tf

```hcl
provider "digitalocean" {
  token = var.do_token
}
```

---

## Step 3.3: Create variables.tf

```hcl
variable "do_token" {
  description = "DigitalOcean API token"
  type        = string
  sensitive   = true
}

variable "ssh_public_key_path" {
  description = "Path to SSH public key"
  type        = string
}

variable "ssh_key_name" {
  description = "Name of the SSH key in DigitalOcean"
  type        = string
  default     = "robin-do-key"
}

variable "droplet_name" {
  description = "Name of the DigitalOcean Droplet"
  type        = string
  default     = "first-ubuntu-24-droplet"
}

variable "region" {
  description = "DigitalOcean region"
  type        = string
  default     = "fra1"
}

variable "droplet_size" {
  description = "DigitalOcean Droplet size"
  type        = string
  default     = "s-1vcpu-512mb-10gb"
}

variable "droplet_image" {
  description = "Droplet image"
  type        = string
  default     = "ubuntu-24-04-x64"
}
```

---

## Step 3.4: Create main.tf

```hcl
resource "digitalocean_ssh_key" "robin_key" {
  name       = var.ssh_key_name
  public_key = file(var.ssh_public_key_path)
}

resource "digitalocean_droplet" "ubuntu_first" {
  name     = var.droplet_name
  region   = var.region
  size     = var.droplet_size
  image    = var.droplet_image
  ssh_keys = [digitalocean_ssh_key.robin_key.fingerprint]
}
```

---

## Step 3.5: Create outputs.tf

```hcl
output "droplet_ip" {
  description = "Public IPv4 address of the Droplet"
  value       = digitalocean_droplet.ubuntu_first.ipv4_address
}

output "ssh_command" {
  description = "SSH command to connect to the Droplet"
  value       = "ssh -i <private-key-path> root@${digitalocean_droplet.ubuntu_first.ipv4_address}"
}
```

---

## Step 3.6: Create terraform.tfvars.example

For Windows Terraform:

```hcl
do_token            = "dop_v1_your_token_here"
ssh_public_key_path = "C:/Users/YOUR_USER/.ssh/robin-do-key.pub"
```

For WSL2 Ubuntu Terraform:

```hcl
do_token            = "dop_v1_your_token_here"
ssh_public_key_path = "/home/YOUR_USER/.ssh/robin-do-key.pub"
```

Important:

* Do not rename `terraform.tfvars.example` to `terraform.tfvars` unless you understand that it may contain secrets.
* Do not submit `terraform.tfvars` if it contains your real API token.
* Prefer using environment variables for the token, as shown in Step 4.

---

## Step 3.7: Optional .gitignore

Create `.gitignore`:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
terraform.tfvars
crash.log
crash.*.log
*.tfplan
```

---

## Step 4: Export API Token

### Windows PowerShell

```powershell
$env:TF_VAR_do_token="dop_v1_your_token_here"
```

### WSL2 Ubuntu

```bash
export TF_VAR_do_token="dop_v1_your_token_here"
```

---

## Step 5: Run Terraform

```bash
terraform init
terraform plan
terraform apply
```

Confirm with:

yes

---

## Step 6: Connect via SSH

### Windows Terminal

```powershell
ssh -i $env:USERPROFILE\.ssh\robin-do-key root@DROPLET_IP
```

### WSL2 Ubuntu

```bash
ssh -i ~/.ssh/robin-do-key root@DROPLET_IP
```

---

## Step 7: Optional PuTTY Access

1. Open PuTTYgen
2. Load private key robin-do-key
3. Save as .ppk
4. Connect using:
   Host: DROPLET_IP
   User: root

---

## Deliverables

Submit:

* Screenshot of terraform apply success
* Screenshot of Droplet in DigitalOcean
* Screenshot of SSH login
* main.tf file

Do not submit:

* API token
* Private SSH key

---

## Cleanup

```bash
terraform destroy
```

Confirm with:

yes

---

## Notes

* Ensure correct file paths for SSH keys
* Ensure API token has proper permissions
* Use WSL2 for more consistent Linux-based workflow
* If SSH fails, verify IP address and key permissions
