# Budgeting Application Setup Guide

A comprehensive step-by-step guide to configure and deploy the budgeting application using Docker and PostgreSQL.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setup Steps](#setup-steps)
  - [Step 1: Create Docker Volume for PostgreSQL Data](#step-1-create-docker-volume-for-postgresql-data)
  - [Step 2: Create Docker Compose Configuration](#step-2-create-docker-compose-configuration)
  - [Step 3: Build the Docker Image](#step-3-build-the-docker-image)
  - [Step 4: Start the PostgreSQL Container](#step-4-start-the-postgresql-container)
  - [Step 5: Verify Container is Running](#step-5-verify-container-is-running)
  - [Step 6: Create Docker Network](#step-6-create-docker-network)
  - [Step 7: Connect PostgreSQL Container to Network](#step-7-connect-postgresql-container-to-network)
  - [Step 8: Access the PostgreSQL Container (Optional)](#step-8-access-the-postgresql-container-optional)
  - [Step 9: Log into PostgreSQL Instance](#step-9-log-into-postgresql-instance)
  - [Step 10: Initialize Database Tables](#step-10-initialize-database-tables)
  - [Step 11: Create Application Database User](#step-11-create-application-database-user)
  - [Step 12: Grant Database Permissions to Application User](#step-12-grant-database-permissions-to-application-user)
  - [Step 13: Verify User Creation](#step-13-verify-user-creation)
  - [Step 14: Configure Environment Variables](#step-14-configure-environment-variables)
  - [Step 15: Load Environment Variables](#step-15-load-environment-variables)
  - [Step 16: Configure Frontend Environment](#step-16-configure-frontend-environment)
  - [Step 17: Build Application Docker Images](#step-17-build-application-docker-images)
  - [Step 18: Start the Complete Application](#step-18-start-the-complete-application)
  - [Step 19: Verify Application is Running](#step-19-verify-application-is-running)
- [Quick Reference Commands](#quick-reference-commands)
  - [Container Management](#container-management)
  - [Database Access](#database-access)
  - [Network Management](#network-management)
- [Troubleshooting](#troubleshooting)
- [Security Best Practices](#security-best-practices)
- [Additional Resources](#additional-resources)
- [Summary](#summary)

---

## Prerequisites

- Docker installed and running
- Docker Compose installed
- Git (to access repository files)
- OpenSSL (for password generation)

---

## Setup Steps

### Step 1: Create Docker Volume for PostgreSQL Data

Create a named Docker volume to persist PostgreSQL data:

```bash
docker volume create budgeting_app
```

**Purpose**: This volume ensures database data persists even if the container is removed.

---

### Step 2: Create Docker Compose Configuration

Create a `docker-compose.yml` file for the PostgreSQL database with the following configuration:

```yaml
services:
  postgres:
    image: postgres:16
    container_name: postgres_budgeting_app
    environment:
      POSTGRES_USER: n4oDbus3rS8j1P3n
      POSTGRES_PASSWORD: 2WHLlz6bLsrZhmZ5oTUHizw65dJSqJQL
      POSTGRES_DB: budgeting_app
    ports:
      - "5432:5432"
    volumes:
      - budgeting_app:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  budgeting_app:
    external: true
```

**Configuration Details**:
- **Image**: postgres:16
- **Container Name**: postgres_budgeting_app
- **Database User**: n4oDbus3rS8j1P3n
- **Database Password**: 2WHLlz6bLsrZhmZ5oTUHizw65dJSqJQL
- **Database Name**: budgeting_app

**Note**: Generate secure passwords using:
```bash
openssl rand -base64 32 | tr -d "=+/" | cut -c1-32
```

---

### Step 3: Build the Docker Image

Build the Docker images defined in the compose file:

```bash
docker compose build
```

---

### Step 4: Start the PostgreSQL Container

Start the container in detached mode:

```bash
docker compose up -d
```

---

### Step 5: Verify Container is Running

Check that the PostgreSQL container is running:

```bash
docker ps
```

You should see `postgres_budgeting_app` in the list of running containers.

---

### Step 6: Create Docker Network

Create a custom Docker network for container communication:

```bash
docker network create budgetapp-network
```

**Purpose**: This network allows different containers (frontend, backend, database) to communicate with each other using container names as DNS.

---

### Step 7: Connect PostgreSQL Container to Network

Add the PostgreSQL container to the newly created network:

```bash
docker network connect budgetapp-network postgres_budgeting_app
```

---

### Step 8: Access the PostgreSQL Container (Optional)

If you need to access the container's bash shell:

```bash
docker exec -it postgres_budgeting_app /bin/bash
```

To exit: Type `exit` and press Enter.

---

### Step 9: Log into PostgreSQL Instance

Connect to the PostgreSQL database using the admin user:

```bash
docker exec -it postgres_budgeting_app psql -U n4oDbus3rS8j1P3n -d budgeting_app
```

You should now be in the PostgreSQL command prompt.

---

### Step 10: Initialize Database Tables

Run the SQL initialization script to create the required database schema.

**Option A**: Copy the SQL file into the container and execute it:
```bash
docker cp sql_init.sql postgres_budgeting_app:/tmp/
docker exec -it postgres_budgeting_app psql -U n4oDbus3rS8j1P3n -d budgeting_app -f /tmp/sql_init.sql
```

**Option B**: Run commands directly from your local machine:
```bash
docker exec -i postgres_budgeting_app psql -U n4oDbus3rS8j1P3n -d budgeting_app < sql_init.sql
```

**Reference**: [SQL Init Script](https://github.com/tevinric/linux_budgetapp/blob/main/database/sql_init.sql)

---

### Step 11: Create Application Database User

Create a dedicated user for the application with limited privileges (security best practice).

**Step 11a**: Generate a secure password:
```bash
openssl rand -base64 32 | tr -d "=+/" | cut -c1-32
```

**Application User Credentials**:
- **Username**: necessaryjigglypuff
- **Password**: SLdjnSkIinImxamsMxQOTJAOG2CL78dC

**Step 11b**: Create the user in PostgreSQL:

While connected to PostgreSQL (from Step 9), run:

```sql
CREATE USER necessaryjigglypuff WITH ENCRYPTED PASSWORD 'SLdjnSkIinImxamsMxQOTJAOG2CL78dC';
```

---

### Step 12: Grant Database Permissions to Application User

Grant the necessary privileges to the application user:

```sql
GRANT ALL PRIVILEGES ON DATABASE budgeting_app TO necessaryjigglypuff;
```

**Note**: For production environments, consider granting more restrictive permissions (SELECT, INSERT, UPDATE, DELETE) instead of ALL PRIVILEGES.

---

### Step 13: Verify User Creation

Verify that the user was created successfully:

```sql
SELECT * FROM pg_user WHERE usename = 'necessaryjigglypuff';
```

Expected output should show the user details. Exit PostgreSQL by typing `\q` and pressing Enter.

---

### Step 14: Configure Environment Variables

Create/update the `env_sample.sh` file with the database connection details:

```bash
#!/bin/bash
# Backend Environment Variables Configuration Script

# Database Configuration
export DB_HOST="postgres_budgeting_app"
export DB_PORT="5432"
export DB_NAME="budgeting_app"
export DB_USER="necessaryjigglypuff"
export DB_PASSWORD="SLdjnSkIinImxamsMxQOTJAOG2CL78dC"

# Server Configuration
export FRONTEND_PORT=3001
export BACKEND_PORT=5001
```

---

### Step 15: Load Environment Variables

Source the environment file to load the variables into your current shell:

```bash
source env_sample.sh
```

Or using the shorthand:

```bash
. env_sample.sh
```

**Verify** variables are loaded:
```bash
echo $DB_HOST
echo $DB_USER
```

---

### Step 16: Configure Frontend Environment

Update the frontend environment file to point to your backend API:

```bash
nano frontend/.env
```

Or use your preferred text editor. Update the API endpoint URLs to use the correct DNS name or localhost.

**Example**:
```
REACT_APP_API_URL=http://localhost:5001
```

Or if using Docker DNS:
```
REACT_APP_API_URL=http://backend_container_name:5001
```

---

### Step 17: Build Application Docker Images

From the root folder of your application (where the main `docker-compose.yml` is located):

```bash
docker compose build
```

This will build both the frontend and backend Docker images.

---

### Step 18: Start the Complete Application

Start all application services (frontend, backend, database):

```bash
docker compose up -d
```

---

### Step 19: Verify Application is Running

Check that all containers are running:

```bash
docker ps
```

You should see:
- postgres_budgeting_app
- Your frontend container
- Your backend container

Check logs if needed:
```bash
docker compose logs -f
```

---

## Quick Reference Commands

### Container Management
```bash
# View running containers
docker ps

# View all containers (including stopped)
docker ps -a

# Stop all services
docker compose down

# Restart services
docker compose restart

# View logs
docker compose logs -f [service_name]
```

### Database Access
```bash
# Connect to PostgreSQL
docker exec -it postgres_budgeting_app psql -U necessaryjigglypuff -d budgeting_app

# Backup database
docker exec postgres_budgeting_app pg_dump -U n4oDbus3rS8j1P3n budgeting_app > backup.sql

# Restore database
docker exec -i postgres_budgeting_app psql -U n4oDbus3rS8j1P3n budgeting_app < backup.sql
```

### Network Management
```bash
# List networks
docker network ls

# Inspect network
docker network inspect budgetapp-network

# Disconnect container from network
docker network disconnect budgetapp-network postgres_budgeting_app
```

---

## Troubleshooting

### Container won't start
- Check logs: `docker logs postgres_budgeting_app`
- Verify port 5432 is not already in use: `netstat -tulpn | grep 5432`

### Cannot connect to database
- Verify container is running: `docker ps`
- Check if container is on the correct network: `docker network inspect budgetapp-network`
- Verify environment variables are set correctly

### Permission denied errors
- Check user privileges in PostgreSQL
- Ensure the application user has the correct grants

### Volume issues
- Verify volume exists: `docker volume ls`
- Inspect volume: `docker volume inspect budgeting_app`

---

## Security Best Practices

1. **Never commit passwords** to version control
2. **Use environment variables** for sensitive data
3. **Generate strong passwords** using cryptographic tools
4. **Limit database user permissions** to only what's needed
5. **Use Docker secrets** in production environments
6. **Keep images updated** regularly for security patches
7. **Use non-root users** in Docker containers when possible

---

## Additional Resources

- [PostgreSQL Docker Setup Guide](https://github.com/tevinric/linux/blob/main/linux_postgres_guide_1.md)
- [Docker Compose Configuration](https://github.com/tevinric/linux/blob/main/linux_postgres_dockercompose_2.yml)
- [SQL Initialization Script](https://github.com/tevinric/linux_budgetapp/blob/main/database/sql_init.sql)

---

## Summary

You've successfully:
✅ Created a Docker volume for persistent storage  
✅ Set up a PostgreSQL database in Docker  
✅ Created a custom Docker network  
✅ Initialized the database schema  
✅ Created an application user with proper permissions  
✅ Configured environment variables  
✅ Built and deployed the complete application stack  

Your budgeting application should now be running and accessible!
