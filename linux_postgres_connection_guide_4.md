# pgAdmin Connection Quick Reference

## Connection Scenarios Summary

### Scenario 1: Linux Server (No GUI) - pgAdmin Web via Docker

**Setup:**
```bash
# Create network
docker network create postgres-network

# Run PostgreSQL (if not already running with network)
docker network connect postgres-network postgres16

# Run pgAdmin
docker run --name pgadmin4 \
  -e PGADMIN_DEFAULT_EMAIL=admin@admin.com \
  -e PGADMIN_DEFAULT_PASSWORD=admin123 \
  -p 5050:80 \
  -v pgadmin_data:/var/lib/pgadmin \
  --network postgres-network \
  -d dpage/pgadmin4:latest
```

**Access:**
- URL: `http://localhost:5050` or `http://server-ip:5050`
- Email: `admin@admin.com`
- Password: `admin123`

**PostgreSQL Connection Settings:**
- Host: `postgres16` ← **Important: Use container name, NOT localhost**
- Port: `5432`
- Database: `mydatabase`
- Username: `myuser`
- Password: `mypassword`

---

### Scenario 2: Development Machine (GUI) - pgAdmin Desktop

**Setup:**
```bash
# Install from: https://www.pgadmin.org/download/

# Or Linux:
sudo apt install pgadmin4-desktop
```

**Access:**
- Launch pgAdmin from applications menu
- Opens in browser (localhost interface)

**PostgreSQL Connection Settings:**
- Host: `localhost` or `127.0.0.1` ← **Important: Use localhost, NOT container name**
- Port: `5432`
- Database: `mydatabase`
- Username: `myuser`
- Password: `mypassword`

**If localhost doesn't work (Windows/Mac Docker Desktop):**
- Host: `host.docker.internal`

---

## Docker Compose with pgAdmin

**File: `docker-compose-with-pgadmin.yml`**

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

**Usage:**
```bash
# Create volume first
docker volume create postgres_data

# Start both services
docker-compose -f docker-compose-with-pgadmin.yml up -d

# Access pgAdmin at http://localhost:5050
# Connect to PostgreSQL using host: postgres16
```

---

## Key Differences

| Aspect | pgAdmin Web (Docker) | pgAdmin Desktop |
|--------|---------------------|-----------------|
| **PostgreSQL Host** | `postgres16` (container name) | `localhost` or `127.0.0.1` |
| **Access Method** | Web browser at port 5050 | Desktop application |
| **Installation** | Docker container | Native application |
| **Network Required** | Yes (Docker network) | No |
| **Remote Access** | Easy (port forwarding) | Needs SSH tunnel |
| **Best For** | Servers, remote management | Local development |

---

## Common Issues & Solutions

### Issue: "Unable to connect to server" (pgAdmin Web)

**Solution:**
- ✓ Use `postgres16` as host (container name)
- ✗ Don't use `localhost` or `127.0.0.1`
- Verify both containers on same network:
  ```bash
  docker network inspect postgres-network
  ```

### Issue: "Connection timeout" (pgAdmin Desktop)

**Solution:**
- ✓ Use `localhost` or `127.0.0.1` as host
- ✗ Don't use container name `postgres16`
- Try `host.docker.internal` on Windows/Mac
- Verify port is exposed:
  ```bash
  docker port postgres16
  ```

### Issue: Can't access pgAdmin at http://localhost:5050

**Solution:**
```bash
# Check if pgAdmin is running
docker ps | grep pgadmin

# Check logs
docker logs pgadmin4

# Restart pgAdmin
docker restart pgadmin4

# Verify port isn't blocked
sudo lsof -i :5050  # Linux/Mac
netstat -ano | findstr :5050  # Windows
```

---

## Quick Commands Reference

### pgAdmin Web Management

```bash
# Start pgAdmin
docker start pgadmin4

# Stop pgAdmin
docker stop pgadmin4

# View logs
docker logs pgadmin4
docker logs -f pgadmin4  # Follow logs

# Restart pgAdmin
docker restart pgadmin4

# Remove pgAdmin (keeps data)
docker stop pgadmin4
docker rm pgadmin4

# Remove pgAdmin and data
docker stop pgadmin4
docker rm pgadmin4
docker volume rm pgadmin_data

# Access pgAdmin shell
docker exec -it pgadmin4 /bin/sh
```

### Network Management

```bash
# Create network
docker network create postgres-network

# List networks
docker network ls

# Inspect network
docker network inspect postgres-network

# Connect existing container to network
docker network connect postgres-network postgres16

# Disconnect container from network
docker network disconnect postgres-network postgres16

# Remove network
docker network rm postgres-network
```

### Testing Connectivity

```bash
# Test from pgAdmin container to PostgreSQL
docker exec -it pgadmin4 ping postgres16

# Test PostgreSQL port from host
nc -zv localhost 5432
telnet localhost 5432

# Test from within PostgreSQL container
docker exec -it postgres16 psql -U myuser -d mydatabase -c "SELECT version();"
```

---

## Security Best Practices

### pgAdmin Web (Production)

```bash
# Use strong credentials
docker run --name pgadmin4 \
  -e PGADMIN_DEFAULT_EMAIL=secure-email@yourdomain.com \
  -e PGADMIN_DEFAULT_PASSWORD=VeryStrongPassword123! \
  -p 5050:80 \
  -v pgadmin_data:/var/lib/pgadmin \
  --network postgres-network \
  -d dpage/pgadmin4:latest
```

**Additional Security:**
- Don't expose port 5050 to internet directly
- Use reverse proxy with HTTPS (Nginx/Apache)
- Use firewall rules to restrict access
- Enable VPN for remote access
- Regular updates: `docker pull dpage/pgadmin4:latest`

### pgAdmin Desktop (Production)

- Set master password for stored credentials
- Don't save passwords for production databases
- Use SSH tunneling for remote connections:
  ```bash
  ssh -L 5432:localhost:5432 user@production-server
  ```
- Keep pgAdmin updated via package manager

---

## Complete Setup Example

### For Headless Linux Server

```bash
# 1. Pull images
docker pull postgres:16
docker pull dpage/pgadmin4:latest

# 2. Create volume and network
docker volume create postgres_data
docker network create postgres-network

# 3. Run PostgreSQL
docker run --name postgres16 \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=mydatabase \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  --network postgres-network \
  -d postgres:16

# 4. Run pgAdmin
docker run --name pgadmin4 \
  -e PGADMIN_DEFAULT_EMAIL=admin@admin.com \
  -e PGADMIN_DEFAULT_PASSWORD=admin123 \
  -p 5050:80 \
  -v pgadmin_data:/var/lib/pgadmin \
  --network postgres-network \
  -d dpage/pgadmin4:latest

# 5. Open browser: http://server-ip:5050
# 6. Login with admin@admin.com / admin123
# 7. Add server with host: postgres16
```

### For Development Machine

```bash
# 1. Install pgAdmin Desktop
# Download from: https://www.pgadmin.org/download/

# 2. Run PostgreSQL
docker run --name postgres16 \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=mydatabase \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  -d postgres:16

# 3. Launch pgAdmin Desktop
# 4. Add server with host: localhost
```

---

## Connection String Examples

### From Application on Host

```
postgresql://myuser:mypassword@localhost:5432/mydatabase
```

### From Application in Docker Container (same network)

```
postgresql://myuser:mypassword@postgres16:5432/mydatabase
```

### Connection String with All Parameters

```
postgresql://myuser:mypassword@localhost:5432/mydatabase?sslmode=disable&connect_timeout=10
```

### Python (psycopg2) Example

```python
import psycopg2

# From host
conn = psycopg2.connect(
    host="localhost",
    port=5432,
    database="mydatabase",
    user="myuser",
    password="mypassword"
)

# From Docker container (same network)
conn = psycopg2.connect(
    host="postgres16",
    port=5432,
    database="mydatabase",
    user="myuser",
    password="mypassword"
)
```

---

## Useful Links

- pgAdmin Documentation: https://www.pgadmin.org/docs/
- PostgreSQL Documentation: https://www.postgresql.org/docs/
- Docker Networks: https://docs.docker.com/network/
- pgAdmin Docker Image: https://hub.docker.com/r/dpage/pgadmin4
