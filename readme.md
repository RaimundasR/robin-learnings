# Advanced Lab: Provision DigitalOcean Droplet with Terraform, Configure Docker with Ansible, and Automate with Bash

## Objective

Create a higher-level DevOps workflow that uses:

* Terraform to provision a DigitalOcean Ubuntu 24.04 Droplet
* Ansible to install Docker on the Droplet
* Docker to run a web-accessible test container
* Bash wrapper script to automate Terraform and Ansible commands
* Terraform to destroy the Droplet after testing

The final workflow should allow commands such as:

```bash
./provisioner.sh --create robin-droplet
```

and:

```bash
./provisioner.sh --create robin-droplet --tool hello-world
```

Important requirement:

The script must only allow creating a Droplet named:

```text
robin-droplet
```

---

## Prerequisites

Use WSL2 Ubuntu or a Linux environment. WSL2 Ubuntu is recommended.

Required tools:

```bash
terraform version
ansible --version
ssh -V
```

Install required tools if missing:

```bash
sudo apt update
sudo apt install -y unzip curl git openssh-client ansible
```

Install Terraform if missing:

```bash
sudo apt update
sudo apt install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | \
  sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update
sudo apt install -y terraform
```

---

## Step 0: Create DigitalOcean API Token

Open:

```text
https://cloud.digitalocean.com/account/api/tokens
```

Create a new token:

```text
Name: terraform-token
Scope: Read and Write
```

Copy the token and store it securely.

Do not commit or submit the API token.

---

## Step 1: Generate SSH Key

Generate SSH key without email comment flag:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/robin-do-key
```

Fix permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/robin-do-key
chmod 644 ~/.ssh/robin-do-key.pub
```

Expected files:

```text
~/.ssh/robin-do-key
~/.ssh/robin-do-key.pub
```

---

## Step 2: Project Folder Structure

Create project:

```bash
mkdir do-terraform-ansible-docker
cd do-terraform-ansible-docker
```

Recommended structure:

```text
do-terraform-ansible-docker/
├── terraform/
│   ├── versions.tf
│   ├── provider.tf
│   ├── variables.tf
│   ├── main.tf
│   ├── outputs.tf
│   └── terraform.tfvars.example
├── ansible/
│   ├── inventory.ini.example
│   └── install-docker.yml
├── provisioner.sh
├── .gitignore
└── README.md
```

Create folders:

```bash
mkdir -p terraform ansible
```

---

# Part 1: Terraform Droplet Creation

## Step 3: Create Terraform Files

### terraform/versions.tf

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

### terraform/provider.tf

```hcl
provider "digitalocean" {
  token = var.do_token
}
```

---

### terraform/variables.tf

```hcl
variable "do_token" {
  description = "DigitalOcean API token"
  type        = string
  sensitive   = true
}

variable "droplet_name" {
  description = "DigitalOcean Droplet name"
  type        = string

  validation {
    condition     = var.droplet_name == "robin-droplet"
    error_message = "Only droplet name robin-droplet is allowed."
  }
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

variable "ssh_key_name" {
  description = "DigitalOcean SSH key name"
  type        = string
  default     = "robin-do-key"
}

variable "ssh_public_key_path" {
  description = "Local path to SSH public key"
  type        = string
  default     = "~/.ssh/robin-do-key.pub"
}
```

---

### terraform/main.tf

```hcl
locals {
  ssh_public_key_path = pathexpand(var.ssh_public_key_path)
}

resource "digitalocean_ssh_key" "robin_key" {
  name       = var.ssh_key_name
  public_key = file(local.ssh_public_key_path)
}

resource "digitalocean_droplet" "robin" {
  name     = var.droplet_name
  region   = var.region
  size     = var.droplet_size
  image    = var.droplet_image
  ssh_keys = [digitalocean_ssh_key.robin_key.fingerprint]
}
```

---

### terraform/outputs.tf

```hcl
output "droplet_ip" {
  description = "Public IPv4 address of the Droplet"
  value       = digitalocean_droplet.robin.ipv4_address
}

output "ssh_command" {
  description = "SSH command to access the Droplet"
  value       = "ssh -i ~/.ssh/robin-do-key root@${digitalocean_droplet.robin.ipv4_address}"
}
```

---

### terraform/terraform.tfvars.example

```hcl
droplet_name        = "robin-droplet"
ssh_public_key_path = "~/.ssh/robin-do-key.pub"
```

Do not place the DigitalOcean API token in this file.

Use an environment variable instead:

```bash
export TF_VAR_do_token="dop_v1_your_token_here"
```

---

## Step 4: Create Droplet with Terraform

```bash
cd terraform
terraform init
terraform plan -var="droplet_name=robin-droplet"
terraform apply -var="droplet_name=robin-droplet"
```

Confirm with:

```text
yes
```

Get the Droplet IP:

```bash
terraform output droplet_ip
```

Test SSH access:

```bash
ssh -i ~/.ssh/robin-do-key root@DROPLET_PUBLIC_IP
```

---

# Part 2: Ansible Docker Installation and Container Run

## Task 1: Install Docker and Run a Web-Accessible Container

After Terraform creates the Droplet, use Ansible to:

1. Connect to the Droplet over SSH
2. Install Docker
3. Enable and start Docker service
4. Pull and run an HTTP test container
5. Expose it on port 80
6. Print the public IP and URL to access the container in a browser

Note:

Docker `hello-world` is not a web server and cannot be accessed from a browser. For browser access, this lab uses the Docker image:

```text
nginxdemos/hello
```

It displays a simple Hello World web page over HTTP.

---

## Step 5: Create Ansible Inventory Example

### ansible/inventory.ini.example

```ini
[droplets]
DROPLET_PUBLIC_IP ansible_user=root ansible_ssh_private_key_file=~/.ssh/robin-do-key
```

The wrapper script will generate the real inventory file automatically.

---

## Step 6: Create Ansible Playbook

### ansible/install-docker.yml

```yaml
---
- name: Install Docker and run hello-world web container
  hosts: droplets
  become: true
  gather_facts: true

  vars:
    container_name: hello-world-web
    container_image: nginxdemos/hello:latest
    container_port: 80

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install required packages
      ansible.builtin.apt:
        name:
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present

    - name: Install Docker
      ansible.builtin.apt:
        name:
          - docker.io
        state: present

    - name: Enable and start Docker service
      ansible.builtin.systemd:
        name: docker
        enabled: true
        state: started

    - name: Pull hello-world web image
      community.docker.docker_image:
        name: "{{ container_image }}"
        source: pull

    - name: Remove existing hello-world web container if present
      community.docker.docker_container:
        name: "{{ container_name }}"
        state: absent

    - name: Run hello-world web container
      community.docker.docker_container:
        name: "{{ container_name }}"
        image: "{{ container_image }}"
        state: started
        restart_policy: always
        published_ports:
          - "{{ container_port }}:80"

    - name: Print browser access URL
      ansible.builtin.debug:
        msg: "Open in browser: http://{{ ansible_host }}:{{ container_port }}"
```

Install required Ansible Docker collection locally:

```bash
ansible-galaxy collection install community.docker
```

---

## Step 7: Run Ansible Manually

Create inventory file:

```bash
cat > ansible/inventory.ini <<EOF
[droplets]
DROPLET_PUBLIC_IP ansible_user=root ansible_ssh_private_key_file=~/.ssh/robin-do-key ansible_host=DROPLET_PUBLIC_IP
EOF
```

Run playbook:

```bash
ansible-playbook -i ansible/inventory.ini ansible/install-docker.yml
```

Open browser:

```text
http://DROPLET_PUBLIC_IP:80
```

Expected result:

A Hello World web page should be visible in the browser.

---

# Part 3: Destroy Existing Droplet with Terraform

## Task 2: Destroy the Droplet

From the Terraform folder:

```bash
cd terraform
terraform destroy -var="droplet_name=robin-droplet"
```

Confirm with:

```text
yes
```

Verify in DigitalOcean that the Droplet was removed.

---

# Part 4: Bash Wrapper Script

## Task 3: Create provisioner.sh

The Bash wrapper should automate Terraform and Ansible.

Required commands:

```bash
./provisioner.sh --create robin-droplet
```

This command should:

1. Validate that the Droplet name is exactly `robin-droplet`
2. Run Terraform init
3. Run Terraform apply
4. Print the Droplet public IP
5. Print the SSH command

---

```bash
./provisioner.sh --create robin-droplet --tool hello-world
```

This command should:

1. Create the Droplet using Terraform
2. Get the Droplet public IP from Terraform output
3. Generate Ansible inventory
4. Install Docker using Ansible
5. Pull and run the Hello World web container
6. Print the browser URL

---

```bash
./provisioner.sh --destroy robin-droplet
```

This command should:

1. Validate that the Droplet name is exactly `robin-droplet`
2. Destroy the Droplet with Terraform

---

## Step 8: Create provisioner.sh

Create file:

```bash
touch provisioner.sh
chmod +x provisioner.sh
```

Add content:

```bash
#!/usr/bin/env bash

set -euo pipefail

ALLOWED_DROPLET_NAME="robin-droplet"
TERRAFORM_DIR="terraform"
ANSIBLE_DIR="ansible"
SSH_KEY_PATH="$HOME/.ssh/robin-do-key"
INVENTORY_FILE="$ANSIBLE_DIR/inventory.ini"
TOOL=""
ACTION=""
DROPLET_NAME=""

usage() {
  cat <<EOF
Usage:
  $0 --create robin-droplet
  $0 --create robin-droplet --tool hello-world
  $0 --destroy robin-droplet

Options:
  --create NAME       Create droplet. Only robin-droplet is allowed.
  --destroy NAME      Destroy droplet. Only robin-droplet is allowed.
  --tool TOOL         Optional tool. Supported: hello-world
  -h, --help          Show help.
EOF
}

require_command() {
  local cmd="$1"
  if ! command -v "$cmd" >/dev/null 2>&1; then
    echo "Error: required command not found: $cmd"
    exit 1
  fi
}

validate_droplet_name() {
  if [[ "$DROPLET_NAME" != "$ALLOWED_DROPLET_NAME" ]]; then
    echo "Error: only droplet name '$ALLOWED_DROPLET_NAME' is allowed."
    exit 1
  fi
}

validate_environment() {
  require_command terraform
  require_command ssh

  if [[ -z "${TF_VAR_do_token:-}" ]]; then
    echo "Error: TF_VAR_do_token is not set."
    echo "Run: export TF_VAR_do_token=\"dop_v1_your_token_here\""
    exit 1
  fi

  if [[ ! -f "$SSH_KEY_PATH" ]]; then
    echo "Error: SSH private key not found: $SSH_KEY_PATH"
    exit 1
  fi

  if [[ ! -f "$SSH_KEY_PATH.pub" ]]; then
    echo "Error: SSH public key not found: $SSH_KEY_PATH.pub"
    exit 1
  fi
}

terraform_create() {
  echo "Running Terraform create for $DROPLET_NAME..."

  terraform -chdir="$TERRAFORM_DIR" init
  terraform -chdir="$TERRAFORM_DIR" apply \
    -var="droplet_name=$DROPLET_NAME" \
    -var="ssh_public_key_path=$SSH_KEY_PATH.pub" \
    -auto-approve

  DROPLET_IP=$(terraform -chdir="$TERRAFORM_DIR" output -raw droplet_ip)

  echo "Droplet created successfully."
  echo "Public IP: $DROPLET_IP"
  echo "SSH command: ssh -i $SSH_KEY_PATH root@$DROPLET_IP"
}

terraform_destroy() {
  echo "Destroying droplet $DROPLET_NAME..."

  terraform -chdir="$TERRAFORM_DIR" destroy \
    -var="droplet_name=$DROPLET_NAME" \
    -var="ssh_public_key_path=$SSH_KEY_PATH.pub" \
    -auto-approve

  echo "Droplet destroyed successfully."
}

wait_for_ssh() {
  local ip="$1"
  local max_attempts=30
  local attempt=1

  echo "Waiting for SSH to become available..."

  while [[ $attempt -le $max_attempts ]]; do
    if ssh \
      -o StrictHostKeyChecking=no \
      -o UserKnownHostsFile=/dev/null \
      -o ConnectTimeout=5 \
      -i "$SSH_KEY_PATH" \
      root@"$ip" "echo ssh-ready" >/dev/null 2>&1; then
      echo "SSH is ready."
      return 0
    fi

    echo "SSH not ready yet. Attempt $attempt/$max_attempts"
    sleep 5
    attempt=$((attempt + 1))
  done

  echo "Error: SSH did not become ready in time."
  exit 1
}

generate_inventory() {
  local ip="$1"

  mkdir -p "$ANSIBLE_DIR"

  cat > "$INVENTORY_FILE" <<EOF
[droplets]
$ip ansible_host=$ip ansible_user=root ansible_ssh_private_key_file=$SSH_KEY_PATH
EOF

  echo "Generated Ansible inventory: $INVENTORY_FILE"
}

run_ansible_hello_world() {
  local ip="$1"

  require_command ansible-playbook

  if [[ "$TOOL" != "hello-world" ]]; then
    echo "Error: unsupported tool '$TOOL'. Supported tool: hello-world"
    exit 1
  fi

  wait_for_ssh "$ip"
  generate_inventory "$ip"

  echo "Installing Docker and running hello-world web container..."

  ansible-playbook \
    -i "$INVENTORY_FILE" \
    "$ANSIBLE_DIR/install-docker.yml"

  echo "Docker hello-world web container is running."
  echo "Browser URL: http://$ip:80"
}

parse_args() {
  if [[ $# -eq 0 ]]; then
    usage
    exit 1
  fi

  while [[ $# -gt 0 ]]; do
    case "$1" in
      --create)
        ACTION="create"
        DROPLET_NAME="${2:-}"
        shift 2
        ;;
      --destroy)
        ACTION="destroy"
        DROPLET_NAME="${2:-}"
        shift 2
        ;;
      --tool)
        TOOL="${2:-}"
        shift 2
        ;;
      -h|--help)
        usage
        exit 0
        ;;
      *)
        echo "Error: unknown argument: $1"
        usage
        exit 1
        ;;
    esac
  done
}

main() {
  parse_args "$@"

  if [[ -z "$ACTION" || -z "$DROPLET_NAME" ]]; then
    echo "Error: action and droplet name are required."
    usage
    exit 1
  fi

  validate_droplet_name
  validate_environment

  case "$ACTION" in
    create)
      terraform_create
      DROPLET_IP=$(terraform -chdir="$TERRAFORM_DIR" output -raw droplet_ip)

      if [[ -n "$TOOL" ]]; then
        run_ansible_hello_world "$DROPLET_IP"
      fi
      ;;
    destroy)
      terraform_destroy
      ;;
    *)
      echo "Error: unsupported action: $ACTION"
      usage
      exit 1
      ;;
  esac
}

main "$@"
```

---

## Step 9: Run the Wrapper Script

Set the DigitalOcean API token:

```bash
export TF_VAR_do_token="dop_v1_your_token_here"
```

Create Droplet only:

```bash
./provisioner.sh --create robin-droplet
```

Create Droplet, install Docker, and run Hello World web container:

```bash
./provisioner.sh --create robin-droplet --tool hello-world
```

Destroy Droplet:

```bash
./provisioner.sh --destroy robin-droplet
```

---

## Step 10: Expected Browser Result

After running:

```bash
./provisioner.sh --create robin-droplet --tool hello-world
```

The script should print something similar to:

```text
Public IP: 203.0.113.10
Browser URL: http://203.0.113.10:80
```

Open the URL in a browser:

```text
http://203.0.113.10:80
```

Expected result:

A Hello World web page from the Docker container.

---

## Step 11: .gitignore

Create `.gitignore`:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
terraform.tfvars
crash.log
crash.*.log
*.tfplan
ansible/inventory.ini
```

---

## Deliverables

Submit:

1. Project folder structure screenshot
2. Terraform files
3. Ansible playbook
4. provisioner.sh script
5. Screenshot of successful Terraform create
6. Screenshot of successful Ansible Docker installation
7. Screenshot of browser showing Hello World page
8. Screenshot of Terraform destroy success

Do not submit:

* DigitalOcean API token
* Private SSH key
* terraform.tfstate file
* terraform.tfvars file containing secrets

---

## Validation Checklist

The solution is complete when:

* Terraform creates only a Droplet named robin-droplet
* SSH key is added to DigitalOcean using Terraform
* The Droplet is Ubuntu 24.04 LTS
* The Droplet uses the smallest specified size
* Ansible installs Docker successfully
* Docker container is accessible over public IP using HTTP
* Wrapper script supports create, create with tool, and destroy
* Terraform destroy removes the Droplet

---

## Common Troubleshooting

### Terraform cannot authenticate

Check:

```bash
echo $TF_VAR_do_token
```

If empty, export the token again:

```bash
export TF_VAR_do_token="dop_v1_your_token_here"
```

---

### SSH permission denied

Check key path:

```bash
ls -la ~/.ssh/robin-do-key ~/.ssh/robin-do-key.pub
```

Fix permissions:

```bash
chmod 600 ~/.ssh/robin-do-key
chmod 644 ~/.ssh/robin-do-key.pub
```

---

### Ansible cannot connect

Test direct SSH:

```bash
ssh -i ~/.ssh/robin-do-key root@DROPLET_PUBLIC_IP
```

If SSH works, test Ansible ping:

```bash
ansible -i ansible/inventory.ini droplets -m ping
```

---

### Browser cannot access container

Check container status on the Droplet:

```bash
ssh -i ~/.ssh/robin-do-key root@DROPLET_PUBLIC_IP

docker ps
curl http://localhost:80
```

If local curl works but browser does not, check firewall or DigitalOcean cloud firewall settings.
