---

# Secure Docker Container Deployment using Ansible Vault

## Project Overview

This project is part of my Ansible and DevOps learning journey.

In this project, I used **Ansible to install Docker on an Ubuntu server and deploy an Nginx Docker container**. I also used **Ansible Vault** to securely store Docker Hub credentials instead of keeping the credentials directly inside the Ansible playbook.

The main purpose of this project was to understand how Ansible, Docker, Docker Hub, and Ansible Vault can work together in a real deployment workflow.

---

## Project Workflow

```text
Linux1 - Ansible Control Node
        |
        | SSH using Ansible
        |
        v
Linux2 - Ubuntu Managed Node
        |
        |-- Docker Installed
        |
        |-- Read Docker Hub Credentials from Ansible Vault
        |
        |-- Login to Docker Hub
        |
        |-- Pull Nginx Image
        |
        |-- Create and Start Nginx Container
        |
        |-- Expose Application on Port 8080
        |
        v
Nginx Container
Port 8080 (Host) --> Port 80 (Container)
```

---

# Technologies Used

* Linux
* Red Hat Enterprise Linux 9.4
* Ubuntu
* Microsoft Azure
* Ansible
* Ansible Roles
* Ansible Vault
* Docker
* Docker Hub
* Nginx
* Git
* GitHub

---

# Lab Environment

For this project, I used two Azure virtual machines.

## Linux1 - Ansible Control Node

* Operating System: Red Hat Enterprise Linux 9.4
* Purpose: Ansible Control Node
* Used for:

  * Running Ansible playbooks
  * Managing the Ubuntu server
  * Storing Ansible Vault
  * Managing Docker deployment

## Linux2 - Managed Node

* Operating System: Ubuntu
* Purpose: Docker Host
* Used for:

  * Installing Docker
  * Logging in to Docker Hub
  * Pulling Docker images
  * Running Docker containers

---

# Project Structure

```text
ansible-docker-vault-deployment/
│
├── docker_installation/
│   └── roles/
│       └── docker/
│           ├── tasks/
│           ├── handlers/
│           ├── vars/
│           ├── defaults/
│           └── ...
│
├── deploy_docker_container.yml
│
├── docker_vault.yml
│
└── README.md
```

---

# Step 1: Install Docker using Ansible

Before deploying the container, Docker needs to be installed on the managed Ubuntu server.

I created an Ansible role for Docker installation.

The role performs tasks such as:

* Updating packages
* Installing Docker
* Adding the required user to the Docker group
* Verifying Docker installation

Example Docker installation task:

```yaml
- name: Install Docker
  ansible.builtin.apt:
    name: docker.io
    state: present
```

---

# Step 2: Add User to Docker Group

After installing Docker, I added the required user to the Docker group.

```yaml
- name: Add users to Docker group
  ansible.builtin.user:
    name: "{{ item }}"
    groups: docker
    append: true
  loop: "{{ docker_users }}"
```

The username is stored in a variable.

For example:

```yaml
docker_users:
  - azureuser
```

This allows the user to run Docker commands without using `sudo` every time.

---

# Step 3: Create Docker Hub Access Token

For Docker Hub authentication, I used a Docker Hub access token.

Instead of storing the Docker Hub credentials directly inside the playbook, I stored them securely using Ansible Vault.

The credentials used in the Vault are:

```yaml
docker_username: "YOUR_DOCKER_HUB_USERNAME"
docker_password: "YOUR_DOCKER_HUB_ACCESS_TOKEN"
```

The actual username and token are not stored in plain text in the GitHub repository.

---

# Step 4: Create Ansible Vault

I created an encrypted Vault file using:

```bash
ansible-vault create docker_vault.yml
```

Inside the Vault file, I added the Docker Hub credentials:

```yaml
docker_username: "YOUR_DOCKER_HUB_USERNAME"
docker_password: "YOUR_DOCKER_HUB_ACCESS_TOKEN"
```

After saving the file, Ansible encrypts the content.

The encrypted file looks similar to this:

```text
$ANSIBLE_VAULT;1.1;AES256
...
```

The encrypted Vault file can be stored in the repository, but the Vault password should never be committed to GitHub.

---

# Step 5: Install the Docker Ansible Collection

The Docker deployment playbook uses modules from the `community.docker` collection.

I installed the collection on the Ansible Control Node.

```bash
ansible-galaxy collection install community.docker
```

To verify the installation:

```bash
ansible-galaxy collection list | grep community.docker
```

---

# Step 6: Install Required Python Package

The Docker Ansible modules require Python dependencies on the managed Docker host.

I installed the required package using Ansible:

```bash
ansible linux2 -b -m ansible.builtin.apt \
-a "name=python3-requests state=present update_cache=yes" -K
```

---

# Step 7: Create Docker Deployment Playbook

The main playbook used in this project is:

```text
deploy_docker_container.yml
```

The playbook performs the following tasks:

1. Reads Docker Hub credentials from Ansible Vault.
2. Logs in to Docker Hub.
3. Pulls the Nginx Docker image.
4. Creates and starts an Nginx container.
5. Maps host port 8080 to container port 80.
6. Displays the running containers.

---

# Docker Deployment Playbook

```yaml
---
- name: Secure Docker Container Deployment
  hosts: linux2
  become: yes

  vars_files:
    - docker_vault.yml

  tasks:

    - name: Login to Docker Hub
      community.docker.docker_login:
        username: "{{ docker_username }}"
        password: "{{ docker_password }}"
      no_log: true

    - name: Pull Nginx Image
      community.docker.docker_image:
        name: nginx:latest
        source: pull

    - name: Run Nginx Container
      community.docker.docker_container:
        name: nginx-container
        image: nginx:latest
        state: started
        restart_policy: unless-stopped
        published_ports:
          - "8080:80"

    - name: Show running containers
      ansible.builtin.command:
        cmd: docker ps
      register: docker_containers

    - name: Display running containers
      ansible.builtin.debug:
        var: docker_containers.stdout_lines
```

---

# Understanding the Playbook

## Reading the Ansible Vault

```yaml
vars_files:
  - docker_vault.yml
```

This tells Ansible to load variables from the encrypted Vault file.

The Vault contains:

```text
docker_username
docker_password
```

---

## Docker Hub Login

```yaml
- name: Login to Docker Hub
  community.docker.docker_login:
    username: "{{ docker_username }}"
    password: "{{ docker_password }}"
  no_log: true
```

The Docker Hub username and access token are taken from Ansible Vault.

I used:

```yaml
no_log: true
```

so that the credentials are not displayed in the Ansible output.

---

## Pull Nginx Image

```yaml
- name: Pull Nginx Image
  community.docker.docker_image:
    name: nginx:latest
    source: pull
```

This task pulls the latest Nginx image from Docker Hub.

---

## Run Nginx Container

```yaml
- name: Run Nginx Container
  community.docker.docker_container:
    name: nginx-container
    image: nginx:latest
    state: started
    restart_policy: unless-stopped
    published_ports:
      - "8080:80"
```

This task creates and starts the Nginx container.

The port mapping:

```text
8080:80
```

means:

```text
Ubuntu Server Port 8080
          |
          v
Docker Container Port 80
          |
          v
Nginx Web Server
```

So the Nginx application can be accessed through port `8080` on the Ubuntu server.

---

# Step 8: Run the Playbook

Before running the playbook, I can check the syntax:

```bash
ansible-playbook deploy_docker_container.yml --syntax-check
```

To run the playbook:

```bash
ansible-playbook deploy_docker_container.yml --ask-vault-pass -K
```

During execution, Ansible asks for:

```text
BECOME password:
```

This is the sudo password used on the managed node.

Then:

```text
Vault password:
```

This is the password used to decrypt the Ansible Vault file.

---

# Step 9: Verify the Docker Container

After the playbook completes successfully, I can check the running containers.

Using Ansible:

```bash
ansible linux2 -b -m command -a "docker ps" -K
```

Or directly on the Ubuntu server:

```bash
docker ps
```

Expected output should show:

```text
nginx-container
```

with a port mapping similar to:

```text
0.0.0.0:8080->80/tcp
```

---

# Step 10: Test the Nginx Application

On the Ubuntu server:

```bash
curl http://localhost:8080
```

If the container is running correctly, Nginx HTML should be returned.

The application can also be accessed from a browser using:

```text
http://SERVER_PUBLIC_IP:8080
```

---

# Azure Network Configuration

Because the Ubuntu server is running on Microsoft Azure, port `8080` must be allowed through the Azure Network Security Group.

An inbound rule should allow:

```text
Protocol: TCP
Port: 8080
Action: Allow
```

For a production environment, the source should be restricted to trusted IP addresses instead of allowing access from everywhere.

---

# Security

One important part of this project was learning how to handle credentials more securely.

## Not Recommended

```yaml
docker_username: "username"
docker_password: "actual-password"
```

inside a normal playbook.

This can expose credentials if the file is uploaded to GitHub.

## Recommended

Store credentials inside an encrypted Ansible Vault:

```text
docker_vault.yml
```

The Vault password should not be committed to GitHub.

Example `.gitignore`:

```text
vault_password.txt
*.retry
```

---

# Git and GitHub

After completing the project, I added the files to my GitHub repository.

Typical workflow:

```bash
git status
```

```bash
git add .
```

```bash
git commit -m "Add secure Docker deployment using Ansible Vault"
```

```bash
git push
```

---

# Problems I Faced During the Project

While working on this project, I faced a few issues. These were useful for understanding how to troubleshoot Ansible and Docker.

## Issue 1: Vault File Not Found

Error:

```text
vars file docker_vault.yml was not found
```

### Reason

The Vault file was not located in the path expected by the playbook.

### Solution

I moved the Vault file into the project directory and used the correct path:

```yaml
vars_files:
  - docker_vault.yml
```

---

## Issue 2: Wrong Docker Container Port Parameter

Error:

```text
Unsupported parameters for community.docker.docker_container module:
publish_ports
```

### Reason

I used:

```yaml
publish_ports
```

instead of the correct parameter:

```yaml
published_ports
```

### Solution

```yaml
published_ports:
  - "8080:80"
```

This was a good reminder to carefully read Ansible module error messages because the supported parameters are often listed in the output.

---

## Issue 3: Docker Service Name

While working on the Docker installation role, I learned that:

```text
docker.io
```

is the package name on Ubuntu, while:

```text
docker
```

is the service name.

For example:

### Package installation

```yaml
name: docker.io
```

### Service management

```yaml
name: docker
```

---

# What I Learned

Through this project, I practiced:

* Installing Docker using Ansible
* Creating and using Ansible Roles
* Managing users and Linux groups
* Using Ansible loops and variables
* Using Ansible Vault for secrets
* Docker Hub authentication
* Pulling Docker images using Ansible
* Creating Docker containers using Ansible
* Docker port mapping
* Using `community.docker` Ansible modules
* Registering command output in Ansible
* Debugging Ansible errors
* Using Git and GitHub to manage DevOps projects

---

# Future Improvements

In the next version of this project, I would like to add:

* Docker Compose deployment
* Multiple containers
* Environment variables using Ansible Vault
* Health checks
* Container monitoring
* Automated container updates
* Deploying a custom application instead of only Nginx
* CI/CD integration using GitHub Actions
* Terraform to provision the infrastructure before Ansible configuration

---

# Conclusion

This project helped me understand a complete basic workflow where Ansible is used to automate Docker deployment and Ansible Vault is used to handle credentials securely.

The complete flow is:

```text
Provisioned Linux Servers
        |
        v
Installed Docker using Ansible
        |
        v
Stored Docker Hub Credentials in Ansible Vault
        |
        v
Logged in to Docker Hub
        |
        v
Pulled Nginx Image
        |
        v
Deployed Nginx Container
        |
        v
Mapped Port 8080 to Container Port 80
        |
        v
Verified Container and Application
```

This project is part of my ongoing hands-on learning in **Linux, Ansible, Docker, Cloud, and DevOps**.

---

## Repository

[View this project on GitHub](https://github.com/Kanishka2201/ansible-learning-lab/tree/main/ansible-docker-vault-deployment)

---

### How to add this README

From your project folder:

```bash
cd /home/azureuser/playbooks/ansible-docker-vault-deployment
```

Open the file:

```bash
nano README.md
```

Replace the existing content with the README above, then save using:

```text
Ctrl + O
Enter
Ctrl + X
```

After that:

```bash
cd /home/azureuser/playbooks
git add ansible-docker-vault-deployment/README.md
git commit -m "Update detailed project documentation"
git push
```

