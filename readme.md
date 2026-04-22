# Terraform + Ansible Task Guide

## Task goal

Provision the **smallest and cheapest DigitalOcean regular droplet** running **Ubuntu 24.04** using **Terraform** with a **best-practice module structure**, then configure the server with **Ansible** to:

* run `apt update`
* install **Docker**
* install **Helm**
* create a non-root user named **`unroot`**

This task teaches a practical DevOps workflow:

1. **Provision infrastructure declaratively** with Terraform.
2. **Separate reusable infrastructure logic into modules**.
3. **Configure the server after creation** with Ansible.
4. **Keep code organized, repeatable, and production-friendly**.

---

## Recommended project structure

```text
infra/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   ├── terraform.tfvars
│   ├── modules/
│   │   └── droplet/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       └── outputs.tf
│   └── inventory.tpl
└── ansible/
    ├── inventory.ini
    ├── playbook.yml
    └── ansible.cfg
```

This is a good baseline because:

* the **root Terraform** layer wires environments together
* the **module** contains reusable droplet logic
* **Ansible** stays separate from provisioning
* outputs can be reused to generate inventory

---

## Terraform

### `terraform/versions.tf`

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    digitalocean = {
      source  = "digitalocean/digitalocean"
      version = "~> 2.0"
    }
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}
```

### `terraform/variables.tf`

```hcl
variable "do_token" {
  description = "DigitalOcean API token"
  type        = string
  sensitive   = true
}

variable "region" {
  description = "DigitalOcean region"
  type        = string
  default     = "fra1"
}

variable "droplet_name" {
  description = "Name of the droplet"
  type        = string
  default     = "ubuntu-24-devops"
}

variable "ssh_keys" {
  description = "List of SSH key fingerprints or IDs already registered in DigitalOcean"
  type        = list(string)
}

variable "tags" {
  description = "Tags for the droplet"
  type        = list(string)
  default     = ["devops", "terraform", "ansible"]
}
```

### `terraform/main.tf`

```hcl
provider "digitalocean" {
  token = var.do_token
}

module "droplet" {
  source = "./modules/droplet"

  name      = var.droplet_name
  region    = var.region
  size      = "s-1vcpu-512mb-10gb"
  image     = "ubuntu-24-04-x64"
  ssh_keys  = var.ssh_keys
  tags      = var.tags
}

resource "local_file" "ansible_inventory" {
  content = templatefile("${path.module}/inventory.tpl", {
    droplet_ip = module.droplet.ipv4_address
  })

  filename = "${path.module}/../ansible/inventory.ini"
}
```

### `terraform/outputs.tf`

```hcl
output "droplet_ip" {
  description = "Public IPv4 address of the droplet"
  value       = module.droplet.ipv4_address
}

output "droplet_id" {
  description = "ID of the droplet"
  value       = module.droplet.id
}
```

### `terraform/terraform.tfvars`

```hcl
do_token     = "your_digitalocean_token_here"
ssh_keys     = ["your_ssh_key_fingerprint_or_id"]
region       = "fra1"
droplet_name = "ubuntu-24-devops"
```

> Do not commit real tokens to Git. In real work, use environment variables or a secrets manager.

### `terraform/inventory.tpl`

```ini
[servers]
app ansible_host=${droplet_ip} ansible_user=root
```

---

## Terraform module

### `terraform/modules/droplet/variables.tf`

```hcl
variable "name" {
  type = string
}

variable "region" {
  type = string
}

variable "size" {
  type = string
}

variable "image" {
  type = string
}

variable "ssh_keys" {
  type = list(string)
}

variable "tags" {
  type    = list(string)
  default = []
}
```

### `terraform/modules/droplet/main.tf`

```hcl
resource "digitalocean_droplet" "this" {
  name     = var.name
  region   = var.region
  size     = var.size
  image    = var.image
  ssh_keys = var.ssh_keys
  tags     = var.tags
}
```

### `terraform/modules/droplet/outputs.tf`

```hcl
output "id" {
  value = digitalocean_droplet.this.id
}

output "ipv4_address" {
  value = digitalocean_droplet.this.ipv4_address
}
```

---

## Ansible

### `ansible/ansible.cfg`

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
retry_files_enabled = False
```

### `ansible/playbook.yml`

```yaml
---
- name: Prepare Ubuntu server
  hosts: servers
  become: true

  vars:
    new_user: unroot

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install required base packages
      ansible.builtin.apt:
        name:
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
          - apt-transport-https
        state: present

    - name: Create user 'unroot'
      ansible.builtin.user:
        name: "{{ new_user }}"
        shell: /bin/bash
        create_home: true
        groups: sudo,docker
        append: true

    - name: Create docker apt keyring directory
      ansible.builtin.file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'

    - name: Add Docker GPG key
      ansible.builtin.shell: |
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
      args:
        creates: /etc/apt/keyrings/docker.gpg

    - name: Add Docker repository
      ansible.builtin.shell: |
        echo \
          "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
          $(. /etc/os-release && echo $VERSION_CODENAME) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
      args:
        creates: /etc/apt/sources.list.d/docker.list

    - name: Update apt cache after Docker repo added
      ansible.builtin.apt:
        update_cache: true

    - name: Install Docker packages
      ansible.builtin.apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present

    - name: Ensure Docker service is enabled and running
      ansible.builtin.service:
        name: docker
        state: started
        enabled: true

    - name: Download Helm install script
      ansible.builtin.get_url:
        url: https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
        dest: /tmp/get_helm.sh
        mode: '0700'

    - name: Install Helm
      ansible.builtin.command: /tmp/get_helm.sh
      args:
        creates: /usr/local/bin/helm
```

---

## How to run

### 1. Export the token instead of hardcoding it

```bash
export TF_VAR_do_token="your_digitalocean_token"
```

Then remove `do_token` from `terraform.tfvars`.

### 2. Initialize Terraform

```bash
cd infra/terraform
terraform init
```

### 3. Review the plan

```bash
terraform plan
```

### 4. Apply the infrastructure

```bash
terraform apply -auto-approve
```

This should create:

* one Ubuntu 24.04 droplet
* generated Ansible inventory in `../ansible/inventory.ini`

### 5. Run the Ansible playbook

```bash
cd ../ansible
ansible-playbook playbook.yml
```

---

## What this task demonstrates

This task checks whether you understand:

* how to use a **Terraform provider**
* how to create a **reusable Terraform module**
* how to separate **root configuration** from **module logic**
* how to pass outputs into another toolchain
* how to use **Ansible** for post-provision configuration
* how to keep infrastructure code maintainable

---

## Professional hints and guidance

### 1. Do not mix provisioning and configuration too much

Terraform should create the VM.
Ansible should configure the OS and software.

That separation is important because:

* Terraform is best for **infrastructure state**
* Ansible is best for **server configuration**

### 2. Avoid hardcoded secrets

A common beginner mistake is storing API tokens in `terraform.tfvars` or Git.
Better choices:

* environment variables
* `.tfvars` excluded by `.gitignore`
* Vault / 1Password / cloud secret manager

### 3. Use modules even for small tasks

Even if there is only one droplet now, modules help when later you need:

* multiple environments
* multiple droplets
* reusable patterns
* team collaboration

### 4. Prefer idempotent Ansible tasks

Ansible should be safe to re-run.
That is why:

* `apt` is used instead of raw shell where possible
* `creates:` is used for shell or command tasks
* services are explicitly enabled

### 5. Root login first, then create safer access

DigitalOcean droplets often start with root SSH access using your uploaded key.
Then Ansible creates `unroot` for normal administration.
A next improvement would be:

* add SSH authorized key for `unroot`
* disable password auth
* disable root SSH login

### 6. Cheapest is not always best for real workloads

The smallest regular droplet is suitable for:

* labs
* demos
* training
* lightweight test workloads

It is usually not enough for:

* production Kubernetes workloads
* heavy Docker workloads
* CI runners
* memory-intensive services

---

## Suggested next improvements

After the basic solution works, extend it with:

1. **Firewall rules** in Terraform
2. **Reserved IP** if needed
3. **Cloud-init user_data** for initial bootstrap
4. **SSH key setup for `unroot`**
5. **Docker daemon hardening**
6. **Helm version pinning**
7. **Remote Terraform state**
8. **Separate environments** such as `dev/`, `stage/`, `prod/`

---

## Interview-style explanation you can say out loud

> The goal is to provision infrastructure with Terraform in a modular and maintainable way, then configure the created Ubuntu server using Ansible. I used a root Terraform layer and a reusable droplet module to follow best practices. After provisioning, Terraform renders an inventory file for Ansible, which then updates apt, installs Docker and Helm, and creates a non-root administrative user named `unroot`. This separation follows the standard DevOps principle of keeping infrastructure provisioning and OS configuration as separate concerns.

---

## Important note

For strict production-grade work, I would improve two things in this student version:

* use the official Ansible modules or repository roles more extensively to reduce shell usage
* verify the exact current DigitalOcean slug for the Ubuntu 24.04 image and the smallest regular droplet size in the target region before applying

---

## Minimal deliverable summary

You are expected to deliver:

* Terraform root configuration
* Terraform droplet module
* generated Ansible inventory
* Ansible playbook
* explanation of why Terraform and Ansible are separated
* instructions to initialize, apply, and configure

---

## Optional `.gitignore`

```gitignore
.terraform/
*.tfstate
*.tfstate.*
crash.log
.terraform.lock.hcl
*.retry
terraform.tfvars
```
