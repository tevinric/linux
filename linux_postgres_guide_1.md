# Complete PostgreSQL 16 Docker Guide

This comprehensive guide covers everything from pulling the PostgreSQL 16 image to running it with Docker Compose, setting up users and databases, and connecting with pgAdmin.

---

## Table of Contents
1. [Pull PostgreSQL 16 Image](#step-1-pull-postgresql-16-image)
2. [Choose Your Setup Method](#step-2-choose-your-setup-method)
3. [Method A: Docker Compose (Recommended)](#method-a-docker-compose-recommended)
4. [Method B: Docker Run](#method-b-docker-run-simple-setup)
5. [Verify Container is Running](#step-3-verify-container-is-running)
6. [Connect to PostgreSQL](#step-4-connect-to-postgresql)
7. [Set Up Additional Users](#step-5-set-up-additional-users)
8. [Install and Connect pgAdmin](#step-6-install-and-connect-pgadmin)
9. [Test Your Setup](#step-7-test-your-setup)
10. [Create New Databases and Users](#creating-new-users-and-databases)
11. [Useful Commands](#useful-commands)
12. [Troubleshooting](#troubleshooting)

---

## Step 1: Pull PostgreSQL 16 Image

First, explicitly pull the PostgreSQL version 16 image from Docker Hub to your local machine:

```bash
docker pull postgres:16
```

Verify the image was downloaded:
```bash
docker images | grep postgres
```

You should see `postgres` with tag `16` in the output.

---

## Step 2: Choose Your Setup Method

You can set up PostgreSQL using two methods:

**Method A: Docker Compose (Recommended)**
- Better for persistent data management
- Easier to manage and recreate
- Configuration stored in a file
- Better for production and development

**Method B: Docker Run**
- Simpler for quick testing
- Single command setup
- Good for learning and experimentation

Choose the method that best fits your needs. Both are included below.

---

## Method A: Docker Compose (Recommended)

### Step 2A-1: Create Docker Volume for Data Persistence

Create a Docker volume to ensure your data persists even if the container is removed:

```bash
docker volume create postgres_data
```

Verify the volume was created:
```bash
docker volume ls
```

You should see `postgres_data` in the list.

To inspect the volume details:
```bash
docker volume inspect postgres_data
```

**Why create a volume manually?**
- Data persists even if you delete the container
- Easy to backup and restore
- Can be shared between containers if needed
- Explicit control over data location

### Step 2A-2: Create Docker Compose File

Create a file named `docker-compose.yml` with the following content:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16
    container_name: postgres16
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydatabase
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  postgres_data:
    external: true
```

**Environment Variables Explained:**
- `POSTGRES_USER=myuser`: Sets the PostgreSQL superuser username
- `POSTGRES_PASSWORD=mypassword`: Sets the password for the user (required)
- `POSTGRES_DB=mydatabase`: Creates initial database and grants access to the user

### Step 2A-3: Start PostgreSQL Container

Navigate to the directory containing your `docker-compose.yml` and run:

```bash
docker-compose up -d
```

The `-d` flag runs the container in detached mode (in the background).

---

## Method B: Docker Run (Simple Setup)

Run the container using a single docker run command with environment variables to create a new user and database:

```bash
docker run --name postgres16 \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=mydatabase \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  -d postgres:16
```

**Parameters Explained:**
- `--name postgres16`: Assigns a name to your container for easy reference
- `-e POSTGRES_USER=myuser`: Sets the desired username (will have superuser privileges)
- `-e POSTGRES_PASSWORD=mypassword`: Sets the password for the specified user (required)
- `-e POSTGRES_DB=mydatabase`: Creates a new database with this name and grants access to the user
- `-p 5432:5432`: Maps container's PostgreSQL port (5432) to the same port on your host
- `-v postgres_data:/var/lib/postgresql/data`: Creates and mounts a volume for data persistence
- `-d`: Runs the container in detached mode (in the background)
- `postgres:16`: Specifies the image name and tag to use

---

## Step 3: Verify Container is Running

Check that your container is running:

```bash
docker ps
```

You should see `postgres16` in the list with status "Up" and port `5432` exposed.

To see more details:
```bash
docker ps -a
```

To check the container logs:
```bash
docker logs postgres16
```

---

## Step 4: Connect to PostgreSQL

### Method 1: Connect Using psql Inside the Container

Connect to your database using the PostgreSQL client inside the container:

```bash
docker exec -it postgres16 psql -U myuser -d mydatabase
```

**Command Breakdown:**
- `docker exec`: Execute a command in a running container
- `-it`: Interactive terminal mode
- `postgres16`: Container name
- `psql`: PostgreSQL interactive terminal
- `-U myuser`: Connect as user 'myuser'
- `-d mydatabase`: Connect to database 'mydatabase'

### Basic PostgreSQL Commands

Once connected, you can execute SQL commands:

#### List all users:
```sql
\du
```

#### List all databases:
```sql
\l
```

#### Show current database and user:
```sql
SELECT current_database(), current_user;
```

#### List all tables in current database:
```sql
\dt
```

#### Get help:
```sql
\?
```

#### Exit psql:
```sql
\q
```

### Method 2: Connect Using Local psql Client

If you have PostgreSQL installed locally:

```bash
psql -h localhost -p 5432 -U myuser -d mydatabase
```

Enter the password when prompted: `mypassword`

---

## Step 5: Set Up Additional Users

### Access PostgreSQL Command Line

```bash
docker exec -it postgres16 psql -U myuser -d mydatabase
```

### Create Different Types of Users

#### Create a basic user:
```sql
CREATE USER appuser WITH PASSWORD 'apppass123';
```

#### Create a developer user with full privileges:
```sql
-- Create user
CREATE USER developer WITH PASSWORD 'devpass123';

-- Grant connection to database
GRANT CONNECT ON DATABASE mydatabase TO developer;

-- Grant usage on schema
GRANT USAGE ON SCHEMA public TO developer;

-- Grant all privileges on all tables in public schema
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO developer;

-- Grant all privileges on all sequences (for auto-increment columns)
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO developer;

-- Grant privileges on future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL PRIVILEGES ON TABLES TO developer;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL PRIVILEGES ON SEQUENCES TO developer;
```

#### Create a read-only user:
```sql
-- Create user
CREATE USER readonly WITH PASSWORD 'readonly123';

-- Grant connection
GRANT CONNECT ON DATABASE mydatabase TO readonly;

-- Grant usage on schema
GRANT USAGE ON SCHEMA public TO readonly;

-- Grant SELECT on all tables
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

-- Grant SELECT on future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly;
```

#### Create a superuser:
```sql
CREATE USER superadmin WITH PASSWORD 'superpass123' SUPERUSER;
```

### List All Users and Their Privileges

```sql
\du
```

This will show you all database roles and their attributes.

### Change User Password

```sql
ALTER USER username WITH PASSWORD 'newpassword';
```

### Delete a User

```sql
DROP USER username;
```

### Exit PostgreSQL Shell

```sql
\q
```

---

## Step 6: Install and Connect pgAdmin

You have two options for using pgAdmin to manage your PostgreSQL databases:
- **Option A:** pgAdmin Web (for Linux servers without GUI - access via browser)
- **Option B:** pgAdmin Desktop (for development machines with GUI)

Choose the option that best fits your environment.

---

### Option A: pgAdmin Web (Headless Linux - No GUI Required)

This method is ideal for Linux servers without a graphical interface. You'll run pgAdmin as a Docker container and access it through a web browser.

#### Step A1: Create Docker Network (Optional but Recommended)

Create a Docker network to allow containers to communicate:

```bash
docker network create postgres-network
```

#### Step A2: Update Your PostgreSQL Container to Use the Network

If you're using **Docker Compose**, update your `docker-compose.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16
    container_name: postgres16
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydatabase
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - postgres-network
    restart: unless-stopped

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin123
    ports:
      - "5050:80"
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    networks:
      - postgres-network
    restart: unless-stopped
    depends_on:
      - postgres

volumes:
  postgres_data:
    external: true
  pgadmin_data:

networks:
  postgres-network:
    driver: bridge
```

Then restart your containers:
```bash
docker-compose down
docker-compose up -d
```

If you're using **Docker Run**, stop your existing container and restart it with the network:

```bash
# Stop existing container
docker stop postgres16

# Add container to network (if already running)
docker network connect postgres-network postgres16

# Start container again
docker start postgres16

# Or recreate the container with network
docker rm postgres16
docker run --name postgres16 \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=mydatabase \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  --network postgres-network \
  -d postgres:16
```

#### Step A3: Run pgAdmin Container

Run pgAdmin as a Docker container:

```bash
docker run --name pgadmin4 \
  -e PGADMIN_DEFAULT_EMAIL=admin@admin.com \
  -e PGADMIN_DEFAULT_PASSWORD=admin123 \
  -p 5050:80 \
  -v pgadmin_data:/var/lib/pgadmin \
  --network postgres-network \
  -d dpage/pgadmin4:latest
```

**Parameters Explained:**
- `--name pgadmin4`: Container name
- `-e PGADMIN_DEFAULT_EMAIL`: Email to login to pgAdmin web interface
- `-e PGADMIN_DEFAULT_PASSWORD`: Password to login to pgAdmin
- `-p 5050:80`: Maps pgAdmin's web interface (port 80) to host port 5050
- `-v pgadmin_data:/var/lib/pgadmin`: Persistent storage for pgAdmin settings
- `--network postgres-network`: Connects to the same network as PostgreSQL
- `-d`: Runs in detached mode

#### Step A4: Access pgAdmin Web Interface

1. Open your web browser and navigate to:
   ```
   http://localhost:5050
   ```
   
   Or if accessing from a remote machine:
   ```
   http://your-server-ip:5050
   ```

2. **Login Credentials:**
   - Email: `admin@admin.com`
   - Password: `admin123`

#### Step A5: Connect PostgreSQL to pgAdmin (Web Interface)

Once logged into pgAdmin web interface:

1. **Right-click on "Servers"** in the left panel

2. **Select "Register" > "Server"**

3. **General Tab:**
   - Name: `PostgreSQL Docker`

4. **Connection Tab:**
   - **Host name/address:** `postgres16` (the container name - NOT localhost!)
   - **Port:** `5432`
   - **Maintenance database:** `mydatabase`
   - **Username:** `myuser`
   - **Password:** `mypassword`
   - ✓ **Save password** (recommended)

5. **Click "Save"**

**Important Note for Web Access:** When pgAdmin runs in a Docker container, you MUST use the PostgreSQL container name (`postgres16`) as the hostname, NOT `localhost` or `127.0.0.1`. This is because both containers are on the same Docker network and communicate using container names.

#### Step A6: Verify Connection

You should now see your PostgreSQL server in the left panel. Expand it to see:
- Databases → mydatabase
- Login/Group Roles (all users)
- Schemas, Tables, etc.

#### Managing pgAdmin Container

**Stop pgAdmin:**
```bash
docker stop pgadmin4
```

**Start pgAdmin:**
```bash
docker start pgadmin4
```

**View pgAdmin logs:**
```bash
docker logs pgadmin4
```

**Remove pgAdmin (keeps data):**
```bash
docker stop pgadmin4
docker rm pgadmin4
```

**Remove pgAdmin and data:**
```bash
docker stop pgadmin4
docker rm pgadmin4
docker volume rm pgadmin_data
```

---

### Option B: pgAdmin Desktop (GUI Application)

This method is for development machines with a graphical interface (Windows, Mac, Linux Desktop).

#### Step B1: Install pgAdmin Desktop

**Download and install pgAdmin:**
https://www.pgadmin.org/download/

**Installation by OS:**

**Windows:**
- Download the Windows installer (.exe)
- Run the installer and follow the setup wizard
- pgAdmin will be added to your Start Menu

**Mac:**
- Download the macOS installer (.dmg)
- Open the .dmg file and drag pgAdmin to Applications
- Launch from Applications folder

**Linux (Ubuntu/Debian):**
```bash
# Install the public key for the repository
curl -fsS https://www.pgadmin.org/static/packages_pgadmin_org.pub | sudo gpg --dearmor -o /usr/share/keyrings/packages-pgadmin-org.gpg

# Create the repository configuration file
sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/packages-pgadmin-org.gpg] https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/$(lsb_release -cs) pgadmin4 main" > /etc/apt/sources.list.d/pgadmin4.list'

# Update package lists
sudo apt update

# Install pgAdmin Desktop
sudo apt install pgadmin4-desktop
```

**Linux (Fedora/RHEL/CentOS):**
```bash
# Install the repository RPM
sudo rpm -i https://ftp.postgresql.org/pub/pgadmin/pgadmin4/yum/pgadmin4-redhat-repo-2-1.noarch.rpm

# Install pgAdmin Desktop
sudo yum install pgadmin4-desktop
```

#### Step B2: Launch pgAdmin Desktop

- **Windows:** Start Menu → pgAdmin 4
- **Mac:** Applications → pgAdmin 4
- **Linux:** Applications Menu → pgAdmin 4 or run `pgadmin4` in terminal

The application will open in your default web browser (it's a web-based interface running locally).

#### Step B3: Connect PostgreSQL to pgAdmin Desktop

1. **Right-click on "Servers"** in the left panel

2. **Select "Register" > "Server"**

3. **General Tab:**
   - Name: `PostgreSQL Docker` (or any descriptive name)

4. **Connection Tab:**
   - **Host name/address:** `localhost` (or `127.0.0.1`)
   - **Port:** `5432`
   - **Maintenance database:** `mydatabase`
   - **Username:** `myuser`
   - **Password:** `mypassword`
   - ✓ **Save password** (optional, for convenience)

5. **Advanced Tab (optional):**
   - **DB restriction:** `mydatabase` (to show only this database)

6. **Click "Save"**

#### Step B4: Verify Connection

You should now see your PostgreSQL server in the left panel. Expand it to see:
- Databases → mydatabase
- Login/Group Roles (where you'll see all the users)
- Schemas → public → Tables

#### Troubleshooting Desktop Connection

If connection fails:

**Try using `127.0.0.1` instead of `localhost`:**
- Some systems have DNS resolution issues with localhost

**For Windows/Mac Docker Desktop, try:**
- Host: `host.docker.internal`
- This is a special DNS name that Docker Desktop provides

**Check if PostgreSQL port is accessible:**
```bash
# Test connection
telnet localhost 5432

# Or using nc (netcat)
nc -zv localhost 5432
```

**Verify container is running and port is exposed:**
```bash
docker ps | grep postgres16
```

---

### Connecting with Different Users in pgAdmin

You can create multiple server connections in pgAdmin (both web and desktop) to test different user privileges:

#### Example: Add Read-Only User Connection

1. **Right-click "Servers"** → **Register** → **Server**

2. **General Tab:**
   - Name: `PostgreSQL Docker (Read-Only)`

3. **Connection Tab:**
   - Host: `localhost` (desktop) or `postgres16` (web)
   - Port: `5432`
   - Maintenance database: `mydatabase`
   - Username: `readonly`
   - Password: `readonly123`

4. **Save**

Now you can switch between connections to test different permission levels!

---

### Managing Databases and Tables via pgAdmin

Once connected, you can perform all database operations through pgAdmin's interface:

#### Create a Database

1. **Right-click "Databases"** under your server
2. **Create** → **Database**
3. Enter database name
4. Set owner (optional)
5. **Save**

#### Create a Table

1. Navigate to: **Databases** → **[your database]** → **Schemas** → **public** → **Tables**
2. **Right-click "Tables"** → **Create** → **Table**
3. **General Tab:** Enter table name
4. **Columns Tab:** Add columns with data types
5. **Constraints Tab:** Add primary keys, foreign keys, etc.
6. **Save**

#### Query Tool

1. **Right-click your database** → **Query Tool**
2. Write SQL queries
3. Click **Execute** (F5) to run
4. View results in the output panel

#### Import/Export Data

**Export Data:**
1. Right-click a table → **Import/Export Data**
2. Choose export format (CSV, JSON, etc.)
3. Configure options
4. **OK**

**Import Data:**
1. Right-click a table → **Import/Export Data**
2. Toggle to **Import**
3. Select file
4. Map columns
5. **OK**

#### Backup Database

1. **Right-click database** → **Backup**
2. Set filename and format
3. Configure options
4. **Backup**

#### Restore Database

1. **Right-click database** → **Restore**
2. Select backup file
3. Configure options
4. **Restore**

---

### Comparison: Web vs Desktop pgAdmin

| Feature | pgAdmin Web (Docker) | pgAdmin Desktop |
|---------|---------------------|-----------------|
| **Installation** | Docker container | Native application |
| **Access** | Web browser | Local application |
| **Use Case** | Headless servers, remote access | Local development |
| **Resource Usage** | Runs in container | Runs natively |
| **Multiple Users** | Yes (with authentication) | Single user |
| **Connection Host** | Container name (postgres16) | localhost/127.0.0.1 |
| **Persistence** | Volume (pgadmin_data) | Local filesystem |
| **Remote Access** | Easy (via port forwarding) | Requires VPN/tunnel |

---

### Security Considerations for pgAdmin

#### For pgAdmin Web (Docker):

1. **Change default credentials:**
   ```bash
   docker stop pgadmin4
   docker rm pgadmin4
   
   docker run --name pgadmin4 \
     -e PGADMIN_DEFAULT_EMAIL=your-email@domain.com \
     -e PGADMIN_DEFAULT_PASSWORD=strong-password-here \
     -p 5050:80 \
     -v pgadmin_data:/var/lib/pgadmin \
     --network postgres-network \
     -d dpage/pgadmin4:latest
   ```

2. **Use HTTPS in production:**
   - Set up a reverse proxy (Nginx/Apache) with SSL
   - Don't expose port 5050 directly to the internet

3. **Restrict access:**
   - Use firewall rules
   - Limit to specific IP addresses
   - Use VPN for remote access

4. **Regular updates:**
   ```bash
   docker pull dpage/pgadmin4:latest
   docker stop pgadmin4
   docker rm pgadmin4
   # Recreate container with new image
   ```

#### For pgAdmin Desktop:

1. **Master password:** Set a master password to encrypt stored passwords
2. **Don't save passwords** for production databases
3. **Use SSH tunneling** for remote connections:
   ```bash
   ssh -L 5432:localhost:5432 user@remote-server
   ```
4. **Keep pgAdmin updated** through your package manager

---

### Quick Setup Summary

#### For Linux Server (No GUI):
```bash
# 1. Create network
docker network create postgres-network

# 2. Run PostgreSQL
docker run --name postgres16 \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=mydatabase \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  --network postgres-network \
  -d postgres:16

# 3. Run pgAdmin
docker run --name pgadmin4 \
  -e PGADMIN_DEFAULT_EMAIL=admin@admin.com \
  -e PGADMIN_DEFAULT_PASSWORD=admin123 \
  -p 5050:80 \
  -v pgadmin_data:/var/lib/pgadmin \
  --network postgres-network \
  -d dpage/pgadmin4:latest

# 4. Access at http://localhost:5050
# 5. Connect using host: postgres16
```

#### For Development Machine (GUI):
```bash
# 1. Install pgAdmin Desktop from https://www.pgadmin.org/download/
# 2. Launch pgAdmin
# 3. Register Server with host: localhost
```

---

## Step 7: Test Your Setup

### Create a Test Table

In pgAdmin or via psql, create a test table:

```sql
CREATE TABLE test_users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Insert Test Data

```sql
INSERT INTO test_users (username, email) VALUES 
    ('john_doe', 'john@example.com'),
    ('jane_smith', 'jane@example.com'),
    ('bob_wilson', 'bob@example.com');
```

### Query the Data

```sql
SELECT * FROM test_users;
```

### Test Different User Permissions

Connect as the read-only user:
```bash
docker exec -it postgres16 psql -U readonly -d mydatabase
```

Try to insert data (this should fail):
```sql
INSERT INTO test_users (username, email) VALUES ('hacker', 'hack@example.com');
-- Should get: ERROR: permission denied for table test_users
```

Try to read data (this should work):
```sql
SELECT * FROM test_users;
-- Should return all rows
```

Exit:
```sql
\q
```

---

## Creating New Users and Databases

Once your PostgreSQL container is running with the persistent volume, you can easily create new databases and users. All data will be stored in the same `postgres_data` volume.

### Create a New Database

#### Using PostgreSQL Command Line

```bash
docker exec -it postgres16 psql -U myuser -d postgres
```

Then run:
```sql
-- Create new database
CREATE DATABASE newdb;

-- Create database with specific owner
CREATE DATABASE projectdb OWNER myuser;

-- List all databases
\l

-- Exit
\q
```

#### Using pgAdmin

1. In pgAdmin, right-click on **Databases** under your server
2. Select **Create** > **Database**
3. Enter database name and owner
4. Click **Save**

### Create Users for the New Database

Access PostgreSQL shell:
```bash
docker exec -it postgres16 psql -U myuser -d postgres
```

Create users with different privilege levels:

```sql
-- Create a user with full access to the new database
CREATE USER newdb_admin WITH PASSWORD 'newdb_pass123';
GRANT ALL PRIVILEGES ON DATABASE newdb TO newdb_admin;

-- Connect to the new database to grant schema privileges
\c newdb

-- Grant schema privileges
GRANT ALL PRIVILEGES ON SCHEMA public TO newdb_admin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO newdb_admin;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO newdb_admin;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL PRIVILEGES ON TABLES TO newdb_admin;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL PRIVILEGES ON SEQUENCES TO newdb_admin;

-- Create a read-only user for the new database
CREATE USER newdb_reader WITH PASSWORD 'reader_pass123';
GRANT CONNECT ON DATABASE newdb TO newdb_reader;
GRANT USAGE ON SCHEMA public TO newdb_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO newdb_reader;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO newdb_reader;

-- Verify users
\du

-- Exit
\q
```

### Connect to New Database in pgAdmin

1. In pgAdmin, right-click on **Servers**
2. Select **Register** > **Server**
3. **General Tab:** Name: `PostgreSQL Docker - NewDB`
4. **Connection Tab:**
   - Host: `localhost`
   - Port: `5432`
   - Maintenance database: `newdb`
   - Username: `newdb_admin`
   - Password: `newdb_pass123`
5. Click **Save**

### Managing Multiple Databases

#### List All Databases

```bash
docker exec -it postgres16 psql -U myuser -c "\l"
```

#### Switch Between Databases in psql

```bash
# Connect to specific database
docker exec -it postgres16 psql -U myuser -d database_name

# Or switch after connecting
\c database_name
```

#### Drop a Database (Careful!)

```bash
docker exec -it postgres16 psql -U myuser -d postgres
```

```sql
-- First, disconnect all users
SELECT pg_terminate_backend(pg_stat_activity.pid)
FROM pg_stat_activity
WHERE pg_stat_activity.datname = 'database_to_drop'
  AND pid <> pg_backend_pid();

-- Then drop the database
DROP DATABASE database_to_drop;
```

#### Create Multiple Databases at Once

```bash
docker exec -it postgres16 psql -U myuser -d postgres << EOF
CREATE DATABASE app_db;
CREATE DATABASE test_db;
CREATE DATABASE staging_db;
\l
EOF
```

### Example: Complete Setup for a New Project

```bash
# Access PostgreSQL
docker exec -it postgres16 psql -U myuser -d postgres
```

Then run these SQL commands:

```sql
-- Create database
CREATE DATABASE project_db;

-- Create admin user
CREATE USER project_admin WITH PASSWORD 'proj_admin_pass';
GRANT ALL PRIVILEGES ON DATABASE project_db TO project_admin;

-- Create app user (limited privileges)
CREATE USER project_app WITH PASSWORD 'proj_app_pass';
GRANT CONNECT ON DATABASE project_db TO project_app;

-- Connect to new database
\c project_db

-- Grant privileges to admin
GRANT ALL PRIVILEGES ON SCHEMA public TO project_admin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO project_admin;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO project_admin;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL PRIVILEGES ON TABLES TO project_admin;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL PRIVILEGES ON SEQUENCES TO project_admin;

-- Grant limited privileges to app user
GRANT USAGE ON SCHEMA public TO project_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO project_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO project_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO project_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO project_app;

-- Verify setup
\l
\du
\q
```

---

## User Credentials Summary

| User | Password | Privileges |
|------|----------|------------|
| myuser | mypassword | Superuser (default/initial) |
| appuser | apppass123 | Basic user |
| developer | devpass123 | All privileges on mydatabase |
| readonly | readonly123 | Read-only access |
| superadmin | superpass123 | Superuser |

**⚠️ IMPORTANT:** Change these passwords in production environments!

---

## Useful Commands

### Docker Management

#### For Docker Compose:

**Stop the container:**
```bash
docker-compose down
```

**Stop and remove all data (careful!):**
```bash
docker-compose down -v
```

**Restart the container:**
```bash
docker-compose restart
```

**View logs:**
```bash
docker-compose logs -f
```

**Start container:**
```bash
docker-compose up -d
```

#### For Docker Run:

**Stop the container:**
```bash
docker stop postgres16
```

**Start the container:**
```bash
docker start postgres16
```

**Remove the container:**
```bash
docker rm postgres16
```

**Remove container and volume:**
```bash
docker rm postgres16
docker volume rm postgres_data
```

**Restart the container:**
```bash
docker restart postgres16
```

**View logs:**
```bash
docker logs postgres16
docker logs -f postgres16  # Follow logs in real-time
```

### PostgreSQL Management

**Access PostgreSQL shell:**
```bash
docker exec -it postgres16 psql -U myuser -d mydatabase
```

**Execute a single SQL command:**
```bash
docker exec -it postgres16 psql -U myuser -d mydatabase -c "SELECT version();"
```

**List all databases:**
```bash
docker exec -it postgres16 psql -U myuser -c "\l"
```

**List all users:**
```bash
docker exec -it postgres16 psql -U myuser -c "\du"
```

**Create a backup:**
```bash
# Backup specific database
docker exec -t postgres16 pg_dump -U myuser mydatabase > mydatabase_backup.sql

# Backup all databases
docker exec -t postgres16 pg_dumpall -U myuser > all_databases_backup.sql
```

**Restore from backup:**
```bash
# Restore specific database
docker exec -i postgres16 psql -U myuser -d mydatabase < mydatabase_backup.sql

# Restore all databases
docker exec -i postgres16 psql -U myuser < all_databases_backup.sql
```

**Check PostgreSQL version:**
```bash
docker exec -it postgres16 psql -U myuser -c "SELECT version();"
```

**Check database size:**
```bash
docker exec -it postgres16 psql -U myuser -c "\l+"
```

### Volume Management

**List volumes:**
```bash
docker volume ls
```

**Inspect volume:**
```bash
docker volume inspect postgres_data
```

**Backup volume data:**
```bash
docker run --rm -v postgres_data:/data -v $(pwd):/backup ubuntu tar czf /backup/postgres_backup.tar.gz -C /data .
```

**Restore volume data:**
```bash
docker run --rm -v postgres_data:/data -v $(pwd):/backup ubuntu tar xzf /backup/postgres_backup.tar.gz -C /data
```

---

## Troubleshooting

### Cannot connect from pgAdmin

**1. Check if container is running:**
```bash
docker ps
```

**2. Check container logs:**
```bash
docker logs postgres16
```

**3. Verify port 5432 is not being used by another service:**
```bash
# Windows
netstat -an | findstr 5432

# Linux/Mac
lsof -i :5432
netstat -tuln | grep 5432
```

**4. Try using `127.0.0.1` instead of `localhost` in pgAdmin**

**5. On Windows/Mac, try `host.docker.internal` as the host**

**6. Ensure password is correct and matches the environment variable**

### Permission denied errors

Make sure the user has the correct privileges:

```bash
docker exec -it postgres16 psql -U myuser -d mydatabase
```

```sql
-- Grant necessary permissions
GRANT ALL PRIVILEGES ON DATABASE mydatabase TO username;
GRANT ALL PRIVILEGES ON SCHEMA public TO username;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO username;
```

### Container fails to start

**Check logs for errors:**
```bash
docker logs postgres16
```

**Common issues:**
- Port 5432 already in use
- Volume permission issues
- Invalid environment variables

**Solution for port conflict:**
```bash
# Use a different port
docker run --name postgres16 -p 5433:5432 ...
```

### Cannot connect to database from application

**1. Ensure the container is running:**
```bash
docker ps
```

**2. Test connection from host:**
```bash
docker exec -it postgres16 psql -U myuser -d mydatabase -c "SELECT 1;"
```

**3. Check if application is using correct connection string:**
```
postgresql://myuser:mypassword@localhost:5432/mydatabase
```

**4. If connecting from another Docker container, use container name instead of localhost:**
```
postgresql://myuser:mypassword@postgres16:5432/mydatabase
```

### pgAdmin Web Cannot Connect to PostgreSQL

**Issue: "Unable to connect to server" in pgAdmin web interface**

**1. Verify both containers are on the same network:**
```bash
docker network inspect postgres-network
```
Both `postgres16` and `pgadmin4` should be listed.

**2. Check pgAdmin logs:**
```bash
docker logs pgadmin4
```

**3. Verify you're using the container name as hostname:**
- ✓ Correct: `postgres16`
- ✗ Wrong: `localhost` or `127.0.0.1`

**4. Test network connectivity between containers:**
```bash
docker exec -it pgadmin4 ping postgres16
```

**5. Restart both containers:**
```bash
docker restart postgres16
docker restart pgadmin4
```

### Cannot Access pgAdmin Web Interface

**Issue: "This site can't be reached" when accessing http://localhost:5050**

**1. Check if pgAdmin container is running:**
```bash
docker ps | grep pgadmin
```

**2. Check pgAdmin logs for errors:**
```bash
docker logs pgadmin4
```

**3. Verify port 5050 is not in use:**
```bash
# Linux/Mac
sudo lsof -i :5050

# Windows
netstat -ano | findstr :5050
```

**4. Try accessing with IP address:**
```bash
# Find your IP
ip addr show  # Linux
ipconfig      # Windows

# Access pgAdmin
http://your-ip-address:5050
```

**5. Check firewall rules:**
```bash
# Linux (Ubuntu)
sudo ufw status
sudo ufw allow 5050/tcp

# CentOS/RHEL
sudo firewall-cmd --list-all
sudo firewall-cmd --add-port=5050/tcp --permanent
sudo firewall-cmd --reload
```

### pgAdmin Desktop Cannot Connect to Dockerized PostgreSQL

**Issue: Connection timeout or refused**

**1. Try different hostnames in this order:**
   - `localhost`
   - `127.0.0.1`
   - `host.docker.internal` (Windows/Mac Docker Desktop)

**2. Verify PostgreSQL port is exposed:**
```bash
docker port postgres16
# Should show: 5432/tcp -> 0.0.0.0:5432
```

**3. Test port connectivity:**
```bash
# Using telnet
telnet localhost 5432

# Using nc (netcat)
nc -zv localhost 5432

# Using psql (if installed)
psql -h localhost -p 5432 -U myuser -d mydatabase
```

**4. Check PostgreSQL configuration:**
```bash
docker exec -it postgres16 cat /var/lib/postgresql/data/pg_hba.conf
```
Should contain a line like:
```
host    all    all    0.0.0.0/0    md5
```

**5. Restart PostgreSQL container:**
```bash
docker restart postgres16
```

### Lost data after container restart

**Check if volume is properly configured:**
```bash
docker inspect postgres16 | grep Mounts -A 10
```

The volume should be mounted at `/var/lib/postgresql/data`

### Reset everything and start fresh

```bash
# Stop and remove container
docker stop postgres16
docker rm postgres16

# Remove volume (WARNING: This deletes all data!)
docker volume rm postgres_data

# Start fresh
# Create volume
docker volume create postgres_data

# Restart with docker-compose or docker run
docker-compose up -d
```

---

## Security Best Practices

1. **Change default passwords** before deploying to production
2. **Use strong passwords:**
   - Minimum 12 characters
   - Mix of uppercase, lowercase, numbers, special characters
   - Use a password manager
3. **Create specific users** for different applications with minimal required privileges
4. **Never use superuser accounts** for regular application access
5. **Use environment variables** or secrets management for passwords:
   ```bash
   # Use .env file for docker-compose
   POSTGRES_PASSWORD=${DB_PASSWORD}
   ```
6. **Enable SSL/TLS** for production connections
7. **Regularly backup** your database
8. **Limit network exposure:**
   - Don't expose port 5432 publicly
   - Use firewall rules
   - Use Docker networks for container-to-container communication
9. **Keep PostgreSQL updated** to the latest version
10. **Monitor database logs** for suspicious activity

---

## Volume Benefits and Data Persistence

All databases and users are stored in the `postgres_data` volume, which means:

✅ **Data persists** across container restarts  
✅ **Container can be removed** without losing data  
✅ **Easy to backup** the entire volume  
✅ **Can migrate** to another server by copying the volume  
✅ **Better performance** than bind mounts  
✅ **Managed by Docker** with proper permissions  

To verify your volume is being used:
```bash
docker volume inspect postgres_data
```

---

## Next Steps

- Create your application schema and tables
- Set up regular automated backups (use cron or scheduled tasks)
- Configure connection pooling (like PgBouncer) for production
- Implement monitoring (pg_stat_statements, pgBadger)
- Set up replication for high availability
- Learn about PostgreSQL performance tuning
- Explore PostgreSQL extensions (PostGIS, pg_trgm, etc.)
- Set up pg_dump scheduled backups

---

## Quick Reference

### Common psql Commands
```sql
\l          -- List all databases
\du         -- List all users
\dt         -- List all tables
\d table    -- Describe table structure
\c dbname   -- Connect to database
\q          -- Quit psql
\?          -- Help on psql commands
\h          -- Help on SQL commands
```

### Connection String Format
```
postgresql://username:password@host:port/database
```

### Example Connection Strings
```bash
# Local connection
postgresql://myuser:mypassword@localhost:5432/mydatabase

# From another Docker container
postgresql://myuser:mypassword@postgres16:5432/mydatabase

# With SSL
postgresql://myuser:mypassword@localhost:5432/mydatabase?sslmode=require
```

---

You now have a fully functional PostgreSQL 16 database running in Docker with complete user management, pgAdmin access, and data persistence!
