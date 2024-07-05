
# Ansible Role for Docker Installation

This repository contains an Ansible role for installing Docker on Ubuntu systems. The playbook automates the process of setting up Docker by following these steps:

1. Updating the apt package index.
2. Installing required packages (`ca-certificates` and `curl`).
3. Adding Docker's official GPG key.
4. Adding Docker's repository to the apt sources list.
5. Installing Docker packages (`docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`).

## Prerequisites

- Ansible installed on your control node.
- SSH access to the target Ubuntu machine(s) with sudo privileges.
