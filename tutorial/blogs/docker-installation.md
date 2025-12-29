# Docker, Git and Docker Compose Installation Guide for Linux

This guide helps you install Docker, the leading containerization tool, and Docker Compose, its companion for managing multi-container applications, on Ubuntu and other Linux distributions, including Amazon Linux 2023. 
Additionally Git for the VCS.

### Prerequisites

* Basic understanding of Linux terminal commands
* Root or sudo privileges

### Update the System

Before we begin, ensure your system is up-to-date:

**For Ubuntu:**
```
sudo apt update && sudo apt upgrade -y
```
**For Amazon Linux 2023:**
```
sudo dnf update -y
```

### Setting Up Docker's GPG Key

1. Install required packages (for Ubuntu):

```
sudo apt-get install ca-certificates curl gnupg lsb-release
```

2. Create a directory for Docker keys:

```
sudo install -m 0755 -d /etc/apt/keyrings
```

3. Download and import the Docker GPG key:

```
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

4. Add Docker repository to sources (Ubuntu):

```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

### Installation

**For Ubuntu:**

```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin git
```

**For Amazon Linux 2023:**

1. Install Docker:

```
sudo dnf install -y docker git
```

2. Install Docker Compose:

```
sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Basic Docker Commands (For both Ubuntu and AL2023)

* Check Docker version:

```
docker --version
```

* Start Docker service:

```
sudo systemctl start docker
```

* Set Docker to start at boot:

```
sudo systemctl enable docker
```

* Check Docker service status:

```
sudo systemctl status docker
```

### Conclusion

By following these steps, you should now have Docker and Docker Compose installed on your Linux system. Refer to the official Docker and Docker Compose documentation for further usage and configuration options.

For more Check - [The Quick Desk](https://blog.thequickdesk.tech/) 
