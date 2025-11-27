
# Guide: Pushing a Local Docker Image to Public and Private Docker Hub Repositories

This guide will walk you through the process of pushing a local Docker image to both public and private repositories on Docker Hub using the terminal.

---

## Prerequisites

- **Docker Installed:** [Get Docker](https://docs.docker.com/get-docker/)
- **Docker Hub Account:** [Sign up here](https://hub.docker.com/signup)
- **Terminal/Command Line Access:** Bash, PowerShell, etc.

---

## 1. Build Your Local Docker Image

If you already have a Docker image, skip this step. Otherwise, navigate to your app’s root folder (with your `Dockerfile`) and run:

```bash
docker build -t my-image:latest .
```
- `-t my-image:latest` gives your image a name (`my-image`) and tag (`latest`)
- `.` tells Docker to use the current directory

---

## 2. Log Into Docker Hub

Authenticate your local Docker client with Docker Hub:

```bash
docker login
```
- Enter your Docker Hub username and password when prompted.

---

## 3. Tag Your Image for Docker Hub

Docker images must be tagged with your Docker Hub username and the repository name.

**For a Public Repo**  
_(Docker Hub repositories are public by default unless you explicitly create a private repo.)_

```bash
docker tag my-image:latest yourdockerhubusername/my-public-repo:latest
```

**For a Private Repo**

First, create a private repository on [Docker Hub](https://hub.docker.com/repositories).

Then tag your image:

```bash
docker tag my-image:latest yourdockerhubusername/my-private-repo:latest
```

---

## 4. Push Your Image to Docker Hub

### Push to Public Repository

```bash
docker push yourdockerhubusername/my-public-repo:latest
```

### Push to Private Repository

```bash
docker push yourdockerhubusername/my-private-repo:latest
```

- If your repository is private, make sure you have permission and are logged in.

---

### Example Workflow

Let’s say your Docker Hub username is `janedoe`, and you want to push to:
- `janedoe/demo-public` (public)
- `janedoe/demo-private` (private)

**Tagging:**
```bash
docker tag my-image:latest janedoe/demo-public:latest
docker tag my-image:latest janedoe/demo-private:latest
```

**Pushing:**
```bash
docker push janedoe/demo-public:latest
docker push janedoe/demo-private:latest
```

---

## 5. Verify Your Images on Docker Hub

- Go to [https://hub.docker.com/repositories](https://hub.docker.com/repositories)
- Check if your image appears under your selected repository.

---

## Troubleshooting Tips

- **Permission Denied:** Make sure you are logged in via `docker login` and have access to the repository.
- **Image Not Showing:** Double-check tags and ensure you've pushed to the correct repository.
- **Rate limits (public repos):** Unauthenticated pulls are subject to rate limits. Log in for higher limits.

---

## Summary of Commands

```bash
# Build
docker build -t my-image:latest .

# Login
docker login

# Tag for public repo
docker tag my-image:latest yourdockerhubusername/my-public-repo:latest

# Tag for private repo
docker tag my-image:latest yourdockerhubusername/my-private-repo:latest

# Push to public repo
docker push yourdockerhubusername/my-public-repo:latest

# Push to private repo
docker push yourdockerhubusername/my-private-repo:latest
```

---

## Additional Resources

- [Docker Push Docs](https://docs.docker.com/engine/reference/commandline/push/)
- [Repository Privacy Settings](https://docs.docker.com/docker-hub/repos/#private-repositories)
