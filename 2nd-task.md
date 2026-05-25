# Terraform + Ansible Lab

This lab is separated into individual files/sections so you can copy each file into its own editor tab.

---

# Project Structure

```text
project/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── terraform.tfvars
│   └── outputs.tf
│
└── ansible/
    ├── inventory.ini
    ├── playbook.yml
    └── nginx.conf
```

---

# Terraform Files

## terraform/main.tf

```hcl
terraform {
  required_providers {
    digitalocean = {
      source  = "digitalocean/digitalocean"
      version = "~> 2.86"
    }
  }
}

provider "digitalocean" {
  token = var.do_token
}

data "digitalocean_ssh_key" "default" {
  name = var.ssh_key_name
}

resource "digitalocean_droplet" "docker_nginx" {
  name     = "docker-nginx-lab"
  region   = "fra1"
  size     = "s-2vcpu-4gb"
  image    = "ubuntu-24-04-x64"

  ssh_keys = [data.digitalocean_ssh_key.default.id]

  tags = [
    "terraform",
    "docker",
    "nginx"
  ]
}
```

---

## terraform/variables.tf

```hcl
variable "do_token" {
  type      = string
  sensitive = true
}

variable "ssh_key_name" {
  type = string
}
```

---

## terraform/terraform.tfvars

```hcl
do_token    = "YOUR_DIGITALOCEAN_TOKEN"
ssh_key_name = "YOUR_SSH_KEY_NAME"
```

---

## terraform/outputs.tf

```hcl
output "public_ip" {
  value = digitalocean_droplet.docker_nginx.ipv4_address
}
```

---

# Terraform Commands

```bash
cd terraform

terraform init
terraform validate
terraform plan
terraform apply
```

Get public IP:

```bash
terraform output public_ip
```

---

# Ansible Files

## ansible/inventory.ini

Replace with your Terraform output IP.

```ini
[droplets]
YOUR_PUBLIC_IP ansible_user=root
```

---

## ansible/nginx.conf

```nginx
server {
    listen 80;
    server_name _;

    location /container1/ {
        proxy_pass http://127.0.0.1:8081/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /container2/ {
        proxy_pass http://127.0.0.1:8082/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## ansible/playbook.yml

```yaml
---
- name: Install Docker and configure Nginx reverse proxy
  hosts: droplets
  become: true

  tasks:

    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install packages
      apt:
        name:
          - docker.io
          - docker-compose-plugin
          - nginx
          - python3-pip
        state: present

    - name: Install Docker Python module
      pip:
        name: docker

    - name: Enable Docker service
      systemd:
        name: docker
        enabled: true
        state: started

    - name: Pull nginx image
      docker_image:
        name: nginx
        source: pull

    - name: Run container1
      docker_container:
        name: container1
        image: nginx:latest
        state: started
        restart_policy: always
        published_ports:
          - "8081:80"

    - name: Run container2
      docker_container:
        name: container2
        image: nginx:latest
        state: started
        restart_policy: always
        published_ports:
          - "8082:80"

    - name: Create custom page in container1
      shell: |
        docker exec container1 bash -c 'echo "<h1>Container1 Working</h1>" > /usr/share/nginx/html/index.html'

    - name: Create custom page in container2
      shell: |
        docker exec container2 bash -c 'echo "<h1>Container2 Working</h1>" > /usr/share/nginx/html/index.html'

    - name: Copy nginx reverse proxy config
      copy:
        src: nginx.conf
        dest: /etc/nginx/sites-available/docker-containers

    - name: Enable nginx site
      file:
        src: /etc/nginx/sites-available/docker-containers
        dest: /etc/nginx/sites-enabled/docker-containers
        state: link
        force: true

    - name: Remove default nginx config
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent

    - name: Validate nginx config
      command: nginx -t

    - name: Restart nginx
      systemd:
        name: nginx
        state: restarted
```

---

# Run Ansible

```bash
cd ansible

ansible-playbook -i inventory.ini playbook.yml
```

---

# Validation

Open browser:

```text
http://YOUR_PUBLIC_IP/container1/
```

Expected:

```text
Container1 Working
```

---

```text
http://YOUR_PUBLIC_IP/container2/
```

Expected:

```text
Container2 Working
```

---

# Verification Commands

Check Docker containers:

```bash
docker ps
```

Check nginx:

```bash
systemctl status nginx
```

Check listening ports:

```bash
ss -tulpn
```

---

# Cleanup

Destroy infrastructure:

```bash
cd terraform
terraform destroy
```
