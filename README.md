# Terraform-Ansible-for-AWS-EC2-Automation
Automates AWS infrastructure provisioning with Terraform and configures worker nodes dynamically using Ansible. Terraform deploys EC2 instances, and Ansible updates the inventory dynamically for seamless remote management.  

## Tech Stack: 
Terraform, Ansible, AWS EC2, Bash

## Step 1: Create main.tf to Provision Resources

This Terraform script will:

- Create a Master Node with Ansible pre-installed
- Create a Worker Node
- Generate an Inventory file dynamically
- Ensure Master can SSH into Worker

```main.tf```
```hcl
# Below is the provider which helps in connecting with AWS Account
provider "aws" {
  region = ""
  profile = ""
}

## Instance type
variable "instance_type" {
  default = "t2.micro"
}

## master node tag
variable "masternode_tag" {
  default = "MasterNode"
}

## worker node tag
variable "workernode_tag" {
  default = "WorkerNode"
}

######### outputs #########
output "worker_ip" {
  value = aws_instance.worker_node.public_ip
}

######### Passing key pairs #########

### Generate a Private Key

resource "tls_private_key" "ansible_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

- Creates a 4096-bit RSA private key.
- This key will be used to authenticate SSH connections securely.

### Upload the Public Key to AWS

resource "aws_key_pair" "ansible_key" {
  key_name   = "ansible_key_1"
  public_key = tls_private_key.ansible_key.public_key_openssh
}

- Uploads the generated public key to AWS under the name ```ansible_key_1```.
- This key will be used when launching EC2 instances.

### Save the Private Key Locally

resource "local_file" "private_key" {
  content  = tls_private_key.ansible_key.private_key_pem
  filename = "ansible_key.pem"
  file_permission = "0400"
}

- Stores the private key in a local file named ```ansible_key.pem```.
- Sets read-only permissions (0400) to prevent unauthorized access.

######### Security Groups ##########


## Master node sg
resource "aws_security_group" "master_sg" {
  name        = "master_security_group"
  description = "Allow SSH and Ansible control"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # Change to restrict access
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

## Slave node sg
resource "aws_security_group" "worker_sg" {
  name        = "worker_security_group"
  description = "Accepts SSH & HTTP from Master"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    security_groups = [aws_security_group.master_sg.id]  # Only Master can SSH
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    security_groups = [aws_security_group.master_sg.id]  # HTTP from Master
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}


######### Data source for ami ##########

## data source for ami
data "aws_ami" "amzlnx2_ami" {
  most_recent = true
  owners = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-kernel*"]
  }
}


######### Servers #########

## master node
resource "aws_instance" "master_node" {
  ami             = data.aws_ami.amzlnx2_ami.id
  instance_type   = var.instance_type
  key_name        = aws_key_pair.ansible_key.key_name
  security_groups = [aws_security_group.master_sg.name]

  tags = {
    Name = var.masternode_tag
  }

  # Add this provisioner block here
  provisioner "file" {
    content     = tls_private_key.ansible_key.private_key_pem
    destination = "/home/ec2-user/ansible_key.pem"

    connection {
      type        = "ssh"
      user        = "ec2-user"
      private_key = tls_private_key.ansible_key.private_key_pem
      host        = self.public_ip
    }
  }

  # Replace your existing user_data with this
  user_data = <<-EOF
    #!/bin/bash
    sudo yum update -y
    sudo amazon-linux-extras install ansible2 -y 
    sudo mkdir -p /home/ec2-user/.ssh
    sudo mv /home/ec2-user/ansible_key.pem /home/ec2-user/.ssh/
    sudo chmod 400 /home/ec2-user/.ssh/ansible_key.pem
    sudo chown ec2-user:ec2-user /home/ec2-user/.ssh/ansible_key.pem
    sudo echo "[worker]" > /home/ec2-user/inventory.ini
    echo "${aws_instance.worker_node.private_ip} ansible_ssh_user=ec2-user ansible_ssh_private_key_file=/home/ec2-user/.ssh/ansible_key.pem" >> /home/ec2-user/inventory.ini
  EOF
}



##worker node
resource "aws_instance" "worker_node" {
  ami             = data.aws_ami.amzlnx2_ami.id
  instance_type   = var.instance_type
  key_name        = aws_key_pair.ansible_key.key_name
  security_groups = [aws_security_group.worker_sg.name]

  tags = {
    Name = var.workernode_tag
  }

  user_data = <<-EOF
    #!/bin/bash
    sudo yum update -y
  EOF
}

```

## Step 2: Deploy Terraform

- Initialize Terraform

```bash
terraform init
```

- Validate Configuration

```bash
terraform validate
```

- Apply Configuration

```bash
terraform apply -auto-approve
```

📌 Outputs will show:

Worker Node Public IP

## Step 3: SSH into Master & Verify Inventory

- Navigate to the directory that has your ```ansible_key.pem``` key run:

```bash
chmod 400 ansible_key.pem
```

- SSH into your Master Node:

```bash
ssh -i ansible_key.pem ec2-user@<master-node-public-ip>
```

- Verify that inventory.ini was created:

```bash
cat /home/ec2-user/inventory.ini
```

It should show:

```bash
[worker]
<WORKER_PRIVATE_IP> ansible_ssh_user=ec2-user ansible_ssh_private_key_file=/home/ec2-user/.ssh/ansible_key.pem
```

- Run ansible ping command

```bash
ansible -i /home/ec2-user/inventory.ini all -m ping
```

# Output:
```json
worker-node-ip | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```


## Step 4: Create Ansible Playbook

This playbook will:

- Update the Worker Node
- Install Apache
- Start and Enable Apache

```ansible_playbook.yml```

```yaml
---
- name: Configure Worker Node
  hosts: worker
  become: yes
  tasks:
    - name: Update system
      yum:
        name: "*"
        state: latest

    - name: Install Apache
      yum:
        name: httpd
        state: present

    - name: Start and enable Apache
      service:
        name: httpd
        state: started
        enabled: yes
```

## Step 5: Run the Playbook

On the Master Node, run:

```bash
ansible-playbook -i inventory.ini ansible_playbook.yml
```

✅ This will:
- Update Worker Node
- Install Apache
- Start Apache

## Step 6: Verify Apache is Running

Copy the ```worker_node_publicip``` and paste on your browser


