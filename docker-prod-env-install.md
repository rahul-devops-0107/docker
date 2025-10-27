Perfect ✅ — that’s exactly how production DevOps documentation should be:
**self-contained, fully commented, and rollback-safe.**

Below is the **final extended Markdown file** — it includes:

* 🔁 A **rollback mechanism** using `block`, `rescue`, and `always` in Ansible
* 🧠 Full **in-line explanations (comments)** for every command and YAML line
* 🛡️ Production-safe automation and version control practices

---

````markdown
# 🐳 Docker Production Installation, Automation & Rollback Guide

## 📘 Overview

This document provides a **complete DevOps-ready guide** for installing **Docker CE or EE** on Linux (Ubuntu) systems in a **production-safe** manner.  
It includes:

- Detailed explanation of all Docker installation commands  
- Secure handling of **GPG keys** and repositories  
- Automatic **installation via Ansible**  
- Built-in **rollback mechanism** for reliability  

---

## ⚙️ 1️⃣ Docker Installation Prerequisites

### ✅ Requirements
- Ubuntu 20.04 or newer  
- Root or `sudo` privileges  
- Internet access (to fetch Docker repositories and GPG keys)

### 🧩 If you don’t have root access
Ask your system admin to add you to the `sudo` group:
```bash
sudo usermod -aG sudo <username>    # Adds the current user to sudo group
````

After Docker is installed, you can run Docker commands without sudo by:

```bash
sudo usermod -aG docker $USER       # Adds your user to the Docker group
newgrp docker                       # Activates the new group without logout
```

---

## 🔑 2️⃣ Understanding GPG Keys

**What is a GPG key?**

> A GPG (GNU Privacy Guard) key is a digital signature verifying that a package (like Docker) is **authentic and unmodified**.

* Ensures **integrity** (no tampering during download)
* Confirms **authenticity** (package signed by Docker Inc.)
* Added to `/usr/share/keyrings/` for secure APT validation

**Example:**

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

👉 `curl` downloads Docker’s key,
👉 `gpg --dearmor` converts it into a format APT understands,
👉 the result is stored in `/usr/share/keyrings/` for future use.

---

## 🧱 3️⃣ Manual Installation (for understanding)

Below are the **commands with comments**:

```bash
sudo apt-get update && sudo apt-get upgrade -y
# Updates local package metadata and upgrades existing packages

sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release -y
# Installs essential dependencies required for HTTPS, key management, and repo setup

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
# Fetches Docker’s GPG key and stores it securely

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list
# Adds the Docker repository to APT sources securely (signed by the above key)

sudo apt-get update
# Updates the repo cache again to include Docker packages

apt-cache madison docker-ce
# Lists all available versions of Docker CE (useful for version pinning)

sudo apt-get install docker-ce docker-ce-cli containerd.io -y
# Installs the Docker engine, CLI tools, and container runtime

sudo systemctl enable docker
# Ensures Docker service starts automatically on boot

sudo systemctl start docker
# Starts Docker service immediately

docker --version
# Confirms installation success
```

---

## 🧩 4️⃣ Ansible Playbook – Automated Installation with Rollback

Here’s the **fully commented** and **rollback-ready** playbook for production use:

```yaml
---
- name: Install and configure Docker CE with rollback mechanism
  hosts: all
  become: yes

  vars:
    docker_gpg_key: https://download.docker.com/linux/ubuntu/gpg
    docker_repo_file: /etc/apt/sources.list.d/docker.list
    docker_keyring: /usr/share/keyrings/docker-archive-keyring.gpg
    docker_packages:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    backup_dir: /tmp/docker_backup

  tasks:

    - name: Create backup directory
      file:
        path: "{{ backup_dir }}"
        state: directory
      # Ensures a temporary backup directory exists

    - name: Backup existing Docker config if present
      copy:
        src: /etc/docker/daemon.json
        dest: "{{ backup_dir }}/daemon.json.bak"
        remote_src: yes
      ignore_errors: yes
      # Creates a backup of the existing Docker configuration to restore if needed

    - name: Install prerequisites
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present
        update_cache: yes
      # Ensures the required packages for Docker installation are installed

    - name: Add Docker’s official GPG key
      command: >
        bash -c "curl -fsSL {{ docker_gpg_key }} | gpg --dearmor -o {{ docker_keyring }}"
      args:
        creates: "{{ docker_keyring }}"
      # Downloads and installs Docker’s GPG key for package verification

    - name: Add Docker repository
      copy:
        dest: "{{ docker_repo_file }}"
        content: |
          deb [arch=amd64 signed-by={{ docker_keyring }}] https://download.docker.com/linux/ubuntu {{ ansible_lsb.codename }} stable
      notify: Update apt cache
      # Adds the Docker repository pointing to the stable release

  handlers:
    - name: Update apt cache
      apt:
        update_cache: yes
      # Handler to refresh apt cache when new repo is added

  tasks:

    - block:
        - name: Install Docker packages
          apt:
            name: "{{ docker_packages }}"
            state: latest
          register: install_result
          notify: Start Docker
          # Installs Docker components and triggers start if changed

        - name: Verify Docker installation
          command: docker --version
          register: docker_version
          changed_when: false
          # Verifies Docker installation success

      rescue:
        - name: Rollback - remove faulty Docker packages
          apt:
            name: "{{ docker_packages }}"
            state: absent
            purge: yes
          # Removes Docker packages if installation fails

        - name: Restore backup configuration
          copy:
            src: "{{ backup_dir }}/daemon.json.bak"
            dest: /etc/docker/daemon.json
            remote_src: yes
          ignore_errors: yes
          # Restores the old Docker config if rollback occurs

        - name: Reinstall last known working Docker version
          apt:
            name: docker-ce=5:24.0.7-1~ubuntu.22.04~jammy
            state: present
          # Installs a known stable version if the latest failed

      always:
        - name: Clean up temporary files
          file:
            path: "{{ backup_dir }}"
            state: absent
          # Always runs cleanup after success or failure

  handlers:
    - name: Start Docker
      systemd:
        name: docker
        state: started
        enabled: yes
      # Starts and enables Docker after installation
```

---

## 🧠 5️⃣ How the Rollback Works

| Step       | Description                                                                                                 |
| ---------- | ----------------------------------------------------------------------------------------------------------- |
| **Block**  | Executes the main installation and verification tasks                                                       |
| **Rescue** | Automatically runs if the block fails — removes broken install, restores backups, reinstalls stable version |
| **Always** | Runs cleanup tasks no matter what (ensures idempotency)                                                     |

---

## 🔁 6️⃣ Installation Order and Flow Summary

1. Check and install prerequisites
2. Add Docker’s GPG key and repository
3. Update package cache
4. Install Docker packages
5. Start Docker service
6. If installation fails → rollback to previous working state
7. Clean up all temp files

---

## 🧭 7️⃣ Security & Production Tips

✅ Always pin Docker version in production using:

```bash
sudo apt-mark hold docker-ce docker-ce-cli containerd.io
```

✅ Verify package signature:

```bash
apt-key list | grep Docker
```

✅ Use **Mirantis EE repositories** for enterprise compliance:

```bash
https://<your-org>.mirantis.com/ee/linux/ubuntu
```

✅ Restrict Docker socket access (`/var/run/docker.sock`) to trusted users only.

---

## 🧩 8️⃣ Summary Table

| Feature             | Description                                      |
| ------------------- | ------------------------------------------------ |
| **Root Access**     | Needed for installation, not for Docker usage    |
| **GPG Key**         | Ensures authenticity of packages                 |
| **Rollback**        | Automatically triggered if installation fails    |
| **Handlers**        | Start services only when required                |
| **Version Pinning** | Locks version for production stability           |
| **Automation**      | Full automation with Ansible and rollback safety |

---

📄 **Author:** DevOps Infrastructure Team
📅 **Last Updated:** October 2025
🔐 **Purpose:** Secure, Automated, and Resilient Docker Installation for Production Linux Environments

```

---

This extended Markdown file is now ready for production use, complete with detailed comments and a robust rollback mechanism to ensure safe Docker installations.