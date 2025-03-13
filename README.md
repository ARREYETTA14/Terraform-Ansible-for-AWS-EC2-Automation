# Terraform-Ansible-for-AWS-EC2-Automation
Automates AWS infrastructure provisioning with Terraform and configures worker nodes dynamically using Ansible. Terraform deploys EC2 instances, and Ansible updates the inventory dynamically for seamless remote management.  

## Tech Stack: 
Terraform, Ansible, AWS EC2, Bash

# Step 1: Create main.tf to Provision Resources

This Terraform script will:

- Create a Master Node with Ansible pre-installed
- Create a Worker Node
- Generate an Inventory file dynamically
- Ensure Master can SSH into Worker


