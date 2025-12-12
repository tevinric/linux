# Prequisites:

Ensure linux system is up to date and you have neccessary packages to add the Docker repository

```
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
````

# Step 1: Setup Docker apt repository

Add Docker's official GPG key and repository to your system's source list: 

### 1. Add Docker's official GPG key:
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### 2. Add the repository to APT sources:
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 3. Update the package database with the new Docker packages:
```
sudo apt-get update
```

# Step 2: Install Docker Engine

Install the latest version of Docker Enginer CLI, containerd and Docker Compose plugin

```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

# Step 3: Verify the installation

```
sudo docker run hello-world
```
