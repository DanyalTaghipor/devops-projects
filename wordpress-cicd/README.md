# WordPress CI/CD Deployment with GitHub Actions

This project demonstrates a CI/CD pipeline for deploying a WordPress application on a remote server using GitHub Actions, Docker, and SSH. The pipeline automates the deployment process, ensuring that the latest changes are deployed to the server whenever code is pushed to the `main` branch.

## Project Structure

- `docker-compose.yml`: Defines the Docker services for WordPress and MySQL.
- `.github/workflows/deploy.yml`: GitHub Actions workflow file that automates the deployment process.

## Prerequisites

- A remote server with SSH access, Docker, and Docker Compose installed.
- SSH key pair for secure access to the remote server.
- GitHub repository with the required secrets configured.

## Getting Started

### 1. Set Up SSH Key Pair

Generate an SSH key pair if you don't already have one:

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

Copy the public key to your remote server:

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub username@remote_host
```

### 2. Configure GitHub Secrets

Add the following secrets to your GitHub repository under **Settings > Secrets and variables > Actions**:

- `REMOTE_HOST`: The IP address or hostname of your remote server.
- `SSH_USERNAME`: The SSH username for your remote server.
- `SSH_PRIVATE_KEY`: The private SSH key for connecting to your remote server.

### 3. Set Up the Docker Compose File

Create a `docker-compose.yml` file to define the WordPress and MySQL services:

```yaml
version: '3.8'

services:
  wordpress:
    image: wordpress:latest
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - ./wp-content:/var/www/html/wp-content

  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

### 4. Create the GitHub Actions Workflow

Create a workflow file at `.github/workflows/deploy.yml` to automate the deployment process:

```yaml
name: Deploy WordPress

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Login to DockerHub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker images
        run: docker-compose -f docker-compose.yml build

      - name: Deploy to remote server
        uses: appleboy/ssh-action@v0.1.7
        with:
          host: ${{ secrets.REMOTE_HOST }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /path/to/your/remote/app
            docker compose down --remove-orphans
            git pull origin main
            docker compose up -d
```

### 5. Deploy

Push changes to the `main` branch of your repository to trigger the GitHub Actions workflow. The workflow will:

1. Navigate to the project directory.
2. Stop any running Docker containers related to the project.
3. Pull the latest changes from the repository.
4. Rebuild and start the Docker containers.
