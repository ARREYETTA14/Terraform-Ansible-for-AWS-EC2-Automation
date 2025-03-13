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

######### key pairs #########

## Master Keypair
resource "aws_key_pair" "master_key" {
  key_name   = "master1_key"
  public_key = file("~/.ssh/id_rsa.pub")
}


## slave keypair
resource "aws_key_pair" "worker_key" {
  key_name   = "worker1_key"
  public_key = file("~/.ssh/id_rsa.pub")
}


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
  key_name        = aws_key_pair.master_key.key_name
  security_groups = [aws_security_group.master_sg.name]

  tags = {
    Name = var.masternode_tag
  }

  user_data = <<-EOF
    #!/bin/bash
    sudo yum update -y
    sudo yum install -y ansible
    sudo echo "[worker]" > /home/ec2-user/inventory.ini
    echo "${aws_instance.worker_node.private_ip} ansible_ssh_user=ec2-user ansible_ssh_private_key_file=/home/ec2-user/.ssh/id_rsa" >> /home/ec2-user/inventory.ini
  EOF
}


##worker node
resource "aws_instance" "worker_node" {
  ami             = data.aws_ami.amzlnx2_ami
  instance_type   = var.instance_type
  key_name        = aws_key_pair.worker_key.key_name
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

- SSH into your Master Node:

```bash
ssh -i ~/.ssh/id_rsa ec2-user@<MASTER_PRIVATE_IP>
```

- Verify that inventory.ini was created:

```bash
cat /home/ec2-user/inventory.ini
```

It should show:

```bash
[worker]
<WORKER_PRIVATE_IP> ansible_ssh_user=ec2-user ansible_ssh_private_key_file=/home/ec2-user/.ssh/id_rsa
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


