# Secure Docker Container Deployment using Ansible Vault

This project demonstrates secure Docker container deployment using Ansible and Ansible Vault.

## Technologies Used

- Ansible
- Docker
- Docker Hub
- Ansible Vault
- Ubuntu
- Microsoft Azure

## Project Workflow

Ansible Control Node → Ansible Vault → Docker Host → Docker Hub → Nginx Container

## Features

- Docker Hub credentials stored securely using Ansible Vault
- Docker Hub login using Ansible
- Pull Nginx Docker image
- Deploy and run Nginx container
- Port mapping from host port 8080 to container port 80
- Container verification using Ansible

## Run the Playbook

```bash
ansible-playbook deploy_docker_container.yml --ask-vault-pass -K
