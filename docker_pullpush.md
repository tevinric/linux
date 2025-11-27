# Guide: Pushing and Pulling Docker Images with Public and Private Docker Hub Repositories

This guide will walk you through the process of building, pushing, and pulling local Docker images to and from public and private repositories on Docker Hub using the terminal. It also explains how to add multiple images to the same repository.

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

## 5. Pulling Images from Docker Hub

You can pull any image that exists in a Docker Hub repository to your local machine.

**Public Repository:**

```bash
docker pull yourdockerhubusername/my-public-repo:latest
```

**Private Repository:**

```bash
docker pull yourdockerhubusername/my-private-repo:latest
```

- For private repositories, make sure you are logged in (`docker login`).

**Example:**

```bash
docker pull janedoe/demo-public:latest
docker pull janedoe/demo-private:latest
```

---

## 6. Adding Multiple Images to the Same Repository (Using Tags)

A single repository on Docker Hub can store multiple images by using different tags. This is useful for versioning or differentiating builds.

**Example: You have two images to push to the same private repo**

1. **Build both images:**
    ```bash
    docker build -t my-app:dev .
    docker build -t my-app:prod .
    ```

2. **Tag each image with a unique tag under the same repository:**
    ```bash
    docker tag my-app:dev yourdockerhubusername/my-private-repo:dev
    docker tag my-app:prod yourdockerhubusername/my-private-repo:prod
    ```

3. **Push both images:**
    ```bash
    docker push yourdockerhubusername/my-private-repo:dev
    docker push yourdockerhubusername/my-private-repo:prod
    ```

**You can use as many tags as you want for different versions or purposes:**

- `yourdockerhubusername/my-private-repo:v1`
- `yourdockerhubusername/my-private-repo:v2`
- `yourdockerhubusername/my-private-repo:experiment`

**To pull a specific tag:**
```bash
docker pull yourdockerhubusername/my-private-repo:dev
docker pull yourdockerhubusername/my-private-repo:prod
```

---

### Full Example Workflow

Let’s say your Docker Hub username is `janedoe`, and you want to push two images (`site` and `api`) to the same private repo (`janedoe/demo-private`):

```bash
# Build your images
docker build -t site:latest ./site
docker build -t api:latest ./api

# Tag for private repo with unique tags
docker tag site:latest janedoe/demo-private:site
docker tag api:latest janedoe/demo-private:api

# Push both images to the same repo under different tags
docker push janedoe/demo-private:site
docker push janedoe/demo-private:api
```

**To pull them:**

```bash
docker pull janedoe/demo-private:site
docker pull janedoe/demo-private:api
```

---

## Additional Troubleshooting Tips

- **Permission Denied:** Make sure you are logged in via `docker login` and have access to the repository.
- **Image Not Showing:** Double-check tags and ensure you've pushed to the correct repository.
- **Rate limits (public repos):** Unauthenticated pulls are subject to rate limits. Log in for higher limits.

---

## Summary of Commands

```bash
# Build images
docker build -t my-image:latest .
docker build -t another-image:latest ./another

# Login
docker login

# Tag for public repo
docker tag my-image:latest yourdockerhubusername/my-public-repo:latest

# Tag multiple for private repo with different tags
docker tag my-image:latest yourdockerhubusername/my-private-repo:app1
docker tag another-image:latest yourdockerhubusername/my-private-repo:app2

# Push to public repo
docker push yourdockerhubusername/my-public-repo:latest

# Push to private repo (multiple tags)
docker push yourdockerhubusername/my-private-repo:app1
docker push yourdockerhubusername/my-private-repo:app2

# Pull from Docker Hub
docker pull yourdockerhubusername/my-public-repo:latest
docker pull yourdockerhubusername/my-private-repo:app1
docker pull yourdockerhubusername/my-private-repo:app2
```

---

## Additional Resources

- [Docker Push Docs](https://docs.docker.com/engine/reference/commandline/push/)
- [Docker Pull Docs](https://docs.docker.com/engine/reference/commandline/pull/)
- [Repository Privacy Settings](https://docs.docker.com/docker-hub/repos/#private-repositories)
- [Docker Tagging Best Practices](https://docs.docker.com/engine/reference/commandline/tag/)
