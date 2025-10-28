# 🟣 Deploy SonarQube with PostgreSQL Using Docker and Docker Compose

# 1. 🎯 Objective

This SOP outlines the steps to deploy SonarQube (Community Edition) with PostgreSQL as its backend database using Docker and Docker Compose on a Linux system.
It includes directory preparation, permissions setup, environment configuration, and container deployment.

# 2. 📋 Prerequisites

- 🖥️ A Linux server (RHEL, CentOS, Fedora, Ubuntu, or similar)
- 🔑 Sudo/root access
- 🌐 Internet connectivity to pull Docker images
- 🐳 Docker and Docker Compose installed
- 🌉 External Docker network: sonarnet (will be created if not present)

# 3. 🔄 Update System Packages
```bash
sudo dnf -y update   # For RHEL/CentOS/Fedora
# OR for Ubuntu/Debian
sudo apt update && sudo -y upgrade
```

# 4. 👤 Create DevOps User (if not already present)
```bash
sudo useradd devops
sudo usermod -aG docker devops
newgrp docker
```

# 5. 📂 Directory Setup and Permissions

Create and set up the directory structure for SonarQube and PostgreSQL:
```bash
mkdir -p ~/containers/sonarqube && cd ~/containers/sonarqube
mkdir sonarqube_data sonarqube_logs sonarqube_extensions postgres_data
```
Determine the container’s running user:
```bash
docker run --rm -it --entrypoint /bin/sh sonarqube:community
/ $ id
uid=999(sonarqube) gid=999(sonarqube)
```
Assign correct ownership:
```bash
sudo chown -R 999:999 sonarqube_data sonarqube_logs sonarqube_extensions postgres_data
```
(Optional) Create Docker network if not already available:
```bash
docker network create sonarnet
```

# 6. ⚙️ Environment Configuration

(Optional) Create a `.env` file in the same directory with the following variables:
```
SONAR_JDBC_URL=jdbc:postgresql://postgres:5432/sonarqube
SONAR_JDBC_USERNAME=sonar
SONAR_JDBC_PASSWORD=sonar
POSTGRES_USER=sonar
POSTGRES_PASSWORD=sonar
POSTGRES_DB=sonarqube
```

# 7. 📄 Create Docker Compose File

Create `docker-compose.yml` under `~/containers/sonarqube/`:
```yaml
version: "3.8"

services:
  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    restart: always
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://postgres:5432/sonarqube
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
      SONAR_WEB_JAVAOPTS: "-Xmx512m -Xms512m -XX:+HeapDumpOnOutOfMemoryError"
      SONAR_SEARCH_JAVAOPTS: "-Xms512m -Xmx512m"
    ports:
      - "9000:9000"
    ulimits:
      nofile:
        soft: 65535
        hard: 65535
      nproc: 4096
    volumes:
      - ./sonarqube_data:/opt/sonarqube/data:z
      - ./sonarqube_logs:/opt/sonarqube/logs:z
      - ./sonarqube_extensions:/opt/sonarqube/extensions:z
    networks:
      - sonarnet
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9000/api/system/status || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  postgres:
    image: postgres:15
    container_name: sonarqube-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
      POSTGRES_DB: sonarqube
    volumes:
      - ./postgres_data:/var/lib/postgresql/data:z
    networks:
      - sonarnet
    depends_on:
      sonarqube:
        condition: service_started
    deploy:
      resources:
        limits:
          memory: 8G
          cpus: "4"
          pids: 100
    security_opt:
      - no-new-privileges:true
      - label:type:container_t

networks:
  sonarnet:
    external: true
```

# 8. 🚀 Deploy SonarQube Stack

Run the container stack:
```bash
docker-compose up -d
```
Verify status:
```bash
docker ps
docker-compose ps
```

# 9. 🧩 Post-Deployment Notes

- 🌐 Web Interface: http://<server_ip>:9000
- 🔑 Default Credentials:
  - Username: admin
  - Password: admin
- 📄 SonarQube Data Directory: ~/containers/sonarqube/sonarqube_data/
- 🪵 Logs Directory: ~/containers/sonarqube/sonarqube_logs/
- 🧱 PostgreSQL Data Directory: ~/containers/sonarqube/postgres_data/

Check container logs:
```bash
docker logs -f sonarqube
docker logs -f sonarqube-postgres
```
Health check:
```bash
curl -I http://localhost:9000/api/system/status
```
Restart containers:
```bash
docker-compose restart
```

# 10. 🧹 Maintenance Tips

Backup the data directories regularly:
```bash
tar -czvf sonarqube_backup_$(date +%F).tar.gz sonarqube_data sonarqube_extensions postgres_data
```
Update container images monthly:
```bash
docker-compose pull
docker-compose up -d
```
Monitor disk usage:
```bash
du -sh ~/containers/sonarqube/*
```

# ✅ Summary

This guide provides a production-ready setup for SonarQube using Docker and Docker Compose with:

- Proper directory structure
- Secure permissions and environment variables
- Resource limits and health checks
- PostgreSQL as the backend database
- Ready-to-use configuration for DevOps CI/CD environments

