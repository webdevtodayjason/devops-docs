# Setting Up a Graylog SYSLOG Server

[![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![Graylog](https://img.shields.io/badge/Graylog-6.1.8-brightgreen.svg?style=for-the-badge&logoColor=white)](https://www.graylog.org/)
[![MongoDB](https://img.shields.io/badge/MongoDB-7.0-green.svg?style=for-the-badge&logo=mongodb&logoColor=white)](https://www.mongodb.com/)
[![OpenSearch](https://img.shields.io/badge/OpenSearch-2.13.0-blue.svg?style=for-the-badge&logoColor=white)](https://opensearch.org/)

> A comprehensive guide to setting up a centralized logging system with Graylog, OpenSearch, and MongoDB using Docker Compose.

## 📋 Overview

This guide walks you through deploying a complete Graylog stack for centralized log management. Graylog allows you to collect, parse, and visualize logs from various sources, making troubleshooting and monitoring easier across your infrastructure.

**Stack Components:**
- **Graylog 6.1.8**: Central log management platform
- **OpenSearch 2.13.0**: Search and analytics engine
- **MongoDB 7.0**: Document database for storing metadata

## 🔍 Prerequisites

- A VM with any cloud hosting service (AWS, Google Cloud, Azure, etc.)
- SSH access to your VM
- Minimum 4GB RAM (8GB recommended)
- At least 20GB of available storage

## 🚀 Step 1: Prepare Your Environment

First, establish an SSH connection to your VM:

```bash
ssh username@your-server-ip
```

### Install Docker and Docker Compose

For Ubuntu/Debian systems:

```bash
# Update package index
sudo apt-get update

# Install Docker
sudo apt-get install -y docker.io

# Install curl or wget
sudo apt-get install -y curl
# OR
sudo apt-get install -y wget

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.7/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify installation
docker-compose --version
```

> 💡 **Note**: If you're using a different OS, please refer to the [official Docker documentation](https://docs.docker.com/engine/install/).

## 🏗️ Step 2: Set Up Project Directory

Create and prepare a directory for your Graylog project:

```bash
# Create project directory
sudo mkdir -p /home/syslog

# Set ownership to current user
sudo chown -R $USER:$USER /home/syslog

# Navigate to the directory
cd /home/syslog
```

## 🔐 Step 3: Generate Security Credentials

Graylog requires two security settings:
1. A password secret (used for encryption)
2. A SHA-256 hash of your admin password

Generate these credentials:

```bash
# Generate password secret (96 characters)
pwgen -N 1 -s 96
# If pwgen is not installed: sudo apt-get install -y pwgen

# Generate SHA-256 hash of your desired admin password
# Replace 'YourStrongPassword' with your actual password
echo -n "YourStrongPassword" | shasum -a 256
```

> ⚠️ **Security Warning**: Store these credentials securely! Never share or expose them in public repositories.

## 📄 Step 4: Create Docker Compose Configuration

Create a `docker-compose.yml` file in your project directory:

```bash
nano docker-compose.yml
```

Copy and paste the following configuration:

```yaml
version: '3'
services:
  mongo:
    image: mongo:7.0
    volumes:
      - mongo_data:/data/db
    restart: unless-stopped
    networks:
      - graylog-net
  
  opensearch:
    image: opensearchproject/opensearch:2.13.0
    environment:
      - discovery.type=single-node
      - "OPENSEARCH_JAVA_OPTS=-Xms2g -Xmx2g"
      - cluster.name=graylog
      - plugins.security.disabled=true
    volumes:
      - os_data:/usr/share/opensearch/data
    restart: unless-stopped
    networks:
      - graylog-net
  
  graylog:
    image: graylog/graylog:6.1.8
    environment:
      # Replace these values with your generated credentials
      - GRAYLOG_PASSWORD_SECRET=YOUR_PASSWORD_SECRET_HERE
      - GRAYLOG_ROOT_PASSWORD_SHA2=YOUR_PASSWORD_SHA2_HERE
      # Replace with your server's public IP or hostname
      - GRAYLOG_HTTP_EXTERNAL_URI=http://your-server-ip:9000/
      - GRAYLOG_MONGODB_URI=mongodb://mongo:27017/graylog
      - GRAYLOG_ELASTICSEARCH_HOSTS=http://opensearch:9200
    entrypoint: /usr/bin/tini -- wait-for-it opensearch:9200 -t 120 -- wait-for-it mongo:27017 -t 120 -- /docker-entrypoint.sh
    depends_on:
      - mongo
      - opensearch
    ports:
      - "9000:9000"     # Web interface
      - "1514:1514"     # Syslog TCP
      - "1514:1514/udp" # Syslog UDP
      - "12201:12201"   # GELF TCP
      - "12201:12201/udp" # GELF UDP
    restart: unless-stopped
    networks:
      - graylog-net

networks:
  graylog-net:
    driver: bridge

volumes:
  mongo_data:
    driver: local
  os_data:
    driver: local
```

> ⚙️ **Configuration Notes**:
> - Replace `YOUR_PASSWORD_SECRET_HERE` with the 96-character secret generated earlier
> - Replace `YOUR_PASSWORD_SHA2_HERE` with the SHA-256 hash generated earlier
> - Replace `your-server-ip` with your VM's public IP address or hostname

Save and exit the file (Ctrl+X, then Y, then Enter in nano).

## ▶️ Step 5: Launch the Graylog Stack

Start the Graylog stack with Docker Compose:

```bash
# Start services in detached mode
docker-compose up -d

# Check the status of the services
docker-compose ps
```

This command will:
1. Download the required Docker images (if not already present)
2. Create the necessary networks and volumes
3. Start all three services in the background

## 📊 Step 6: Monitor Service Logs

You can monitor the logs of each service to check for issues and track the startup process:

### Basic log monitoring:

```bash
# View logs for a specific service
docker-compose logs -f graylog
docker-compose logs -f mongo
docker-compose logs -f opensearch
```

### Advanced monitoring with tmux:

For a more comprehensive view, you can use tmux to monitor all services simultaneously:

```bash
# Install tmux if not already installed
sudo apt-get install -y tmux

# Create a new tmux session with multiple windows
tmux new-window -n graylog 'docker-compose logs -f graylog' \; split-window -h -t 0 'docker-compose logs -f mongo' \; split-window -v -t 1 'docker-compose logs -f opensearch'
```

**Tmux navigation tips:**
- Switch between panes: `Ctrl+B`, then arrow keys
- Detach from session: `Ctrl+B`, then `D`
- Reattach to session: `tmux attach`
- Scroll in a pane: `Ctrl+B`, then `[` (use arrow keys to scroll, press `q` to exit scroll mode)

## 🌐 Step 7: Access the Graylog Web Interface

Once all services are running, access the Graylog web interface in your browser:

```
http://your-server-ip:9000
```

Log in with:
- Username: `admin`
- Password: The password you used to generate the SHA-256 hash

![Graylog Login Screen](https://i.imgur.com/example-image.png)

## 🛠️ Step 8: Basic Configuration

After logging in, you'll need to configure Graylog to start receiving logs:

1. **Create an Input**:
   - Navigate to System → Inputs
   - Select "Syslog UDP" from the dropdown
   - Click "Launch new input"
   - Configure with defaults or customize as needed
   - Click "Save"

2. **Create an Index Set** (optional):
   - Navigate to System → Indices
   - Click "Create index set"
   - Set rotation and retention policies according to your needs

3. **Configure Dashboards** (optional):
   - Navigate to Dashboards
   - Click "Create dashboard"
   - Add widgets to visualize your log data

## 🔄 Service Management

Here are some useful commands for managing your Graylog stack:

```bash
# Stop all services
docker-compose down

# Stop and remove volumes (caution: this deletes all data!)
docker-compose down -v

# Stop and remove all images
docker-compose down --rmi all

# Restart all services
docker-compose restart

# Restart a specific service
docker-compose restart graylog
```

## 📱 Configure Client Devices

To send logs from other devices to your Graylog server:

### Linux systems (using rsyslog):

Edit `/etc/rsyslog.conf`:

```bash
# Add this line to send logs to your Graylog server
*.* @your-server-ip:1514
```

Restart rsyslog:

```bash
sudo systemctl restart rsyslog
```

### Network devices:

Configure your network devices to send syslog messages to:
- Server: `your-server-ip`
- Port: `1514`
- Protocol: UDP (or TCP if configured)

## 🔍 Troubleshooting

### Common Issues:

1. **Cannot access the web interface**:
   - Check if the services are running: `docker-compose ps`
   - Verify firewall settings: `sudo ufw status`
   - Ensure port 9000 is open: `sudo ufw allow 9000/tcp`

2. **OpenSearch fails to start**:
   - Check system resources: `free -m` and `df -h`
   - Verify OpenSearch logs: `docker-compose logs opensearch`
   - May need to adjust Java heap size in docker-compose.yml

3. **Not receiving logs**:
   - Verify input is running in Graylog web interface
   - Check firewall for UDP port 1514: `sudo ufw allow 1514/udp`
   - Test with a manual syslog message: `logger -n your-server-ip -P 1514 "Test message"`

## 📚 Further Reading

- [Official Graylog Documentation](https://docs.graylog.org/)
- [OpenSearch Documentation](https://opensearch.org/docs/)
- [MongoDB Documentation](https://docs.mongodb.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

---

[← Back to Main Page](../README.md)
