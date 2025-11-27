Ensure you have Docker installed on your Ubuntu/Linux machine.

### Step 1: Pull the PostgreSQL Image
Open your terminal and download the official PostgreSQL image from Docker Hub.
```bash
docker pull postgres:16
```
- **docker pull postgres:16**: This command downloads (pulls) the official image for PostgreSQL version 16 from Docker Hub so you can use it to create containers.

### Step 2: Create a Docker Volume for Data Persistence
Create a named volume to ensure your data is stored safely on the host machine and is not lost when the container stops or is removed. This volume will physically reside within the Docker managed area.

```bash
docker volume create pg_data_v16
```
- **docker volume create pg_data_v16**: This creates a persistent Docker volume named `pg_data_v16` to store your PostgreSQL data. Volumes are managed areas outside of the container lifecycle.

##### Volume Details and Location:
To find the exact location of this volume on your Linux filesystem, use the following command:

```bash
docker volume inspect pg_data_v16
```
- **docker volume inspect pg_data_v16**: Inspects the details of your volume, including its location (`Mountpoint`) on disk.

### Step 3: Run the PostgreSQL Container and Initialize the Database/User
Run the container using a single command that defines the new user (`myuser`) and database (`mydatabase`) via environment variables. The image configures these on startup.

#### CREATE AND RUN CONTAINER

```bash
docker run -d --name my_postgres_container_v16 \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_DB=mydatabase \
  -v pg_data_v16:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16
```
- **docker run**: Start a new Docker container.
- **-d**: Run the container in detached mode (in the background).
- **--name my_postgres_container_v16**: Assign a name to your container.
- **-e POSTGRES_USER=myuser**: Set the PostgreSQL username to `myuser`.
- **-e POSTGRES_PASSWORD=mysecretpassword**: Set the password for the user.
- **-e POSTGRES_DB=mydatabase**: Name the default database.
- **-v pg_data_v16:/var/lib/postgresql/data**: Mount the persistent volume into the container where Postgres expects to store its data.
- **-p 5432:5432**: Map port 5432 on the host to port 5432 in the container (the default for PostgreSQL).
- **postgres:16**: Use the official PostgreSQL image version 16.

#### START EXISTING CONTAINER
```bash
docker start my_postgres_container_v16
```
- **docker start my_postgres_container_v16**: Start your previously-created container if it stopped.

### Step 4: Verify the Container is Running

```bash
docker ps
```
- **docker ps**: List running containers. You should see your PostgreSQL container if it’s running.

### Step 5: Access the Database and Create a Table

Access the shell inside your container as the user you just created to connect to your database:

```bash
docker exec -it my_postgres_container_v16 psql -U myuser -d mydatabase
```
- **docker exec -it**: Run a command inside an existing container and give you an interactive shell.
- **psql -U myuser -d mydatabase**: Use the PostgreSQL interactive terminal (`psql`) as `myuser`, connecting to `mydatabase`.

Create a table:
```sql
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  price NUMERIC(10, 2)
);
```
- Creates a new table called `products` with fields for the product ID, name, and price.

Insert some data:
```sql
INSERT INTO products (name, price) VALUES ('Laptop', 999.99), ('Mouse', 25.50);
```
- Inserts two products into the `products` table.

Verify the data:
```sql
SELECT * FROM products;
```
- Displays all the data in the `products` table.

Type `\q` and press Enter to exit the `psql` shell.

---

### Step 6: Backing up Your Data

Since all data is stored in the pg_data volume, you can back it up using standard Linux tools targeting the Mountpoint path found in Step 2. A more reliable method for a live database is to use the PostgreSQL `pg_dump` utility inside your container.

#### Create a local directory for backups
```bash
mkdir -p ~/db_backups
```
- **mkdir -p ~/db_backups**: Creates the `db_backups` directory in your home folder. The `-p` flag avoids errors if the directory already exists or if parent folders are missing.

#### Use pg_dump inside the container to create an SQL dump file on your host machine

```bash
docker exec -t my_postgres_container_v16 pg_dump -U myuser mydatabase > ~/db_backups/mydatabase_dump_$(date +\%Y\%m\%d).sql
```
- **docker exec -t**: Runs the command in your container (no interactive shell needed).
- **pg_dump -U myuser mydatabase**: Dumps the database as SQL text.
- **> ~/db_backups/mydatabase_dump_$(date +\%Y\%m\%d).sql**: Saves the output to a file named with today's date.

---

### Step 7: Automate Backups with a Cron Job (Every 5 Minutes, Keep 10 Dumps)

You can use `cron` to automatically back up your PostgreSQL database every 5 minutes and use a simple script to delete older backups, keeping only the 10 most recent ones.

#### 1. Create the backup script

Create a file called `~/db_backups/auto_backup.sh` with the following contents:

```bash
#!/bin/bash

# Dump the database to a timestamped file
docker exec -t my_postgres_container_v16 pg_dump -U myuser mydatabase > ~/db_backups/mydatabase_dump_$(date +\%Y\%m\%d_\%H\%M\%S).sql

# Remove all but the 10 most recent backup files
cd ~/db_backups
ls -1t mydatabase_dump_*.sql | tail -n +11 | xargs -r rm --
```
- **#!/bin/bash**: Interpreter directive—this script should be run with Bash.
- **docker exec ... pg_dump ... > ...**: Creates a timestamped SQL dump file in `db_backups`.
- **cd ~/db_backups**: Change to your backup directory.
- **ls -1t mydatabase_dump_*.sql**: List all backup files, sorted by time (newest first).
- **tail -n +11**: Show all except the first 10 files (i.e., files older than the 10 most recent).
- **xargs -r rm --**: Delete those older files (if any).

Make the script executable:

```bash
chmod +x ~/db_backups/auto_backup.sh
```
- **chmod +x**: Adds execute permissions so you can run the script.

#### 2. Add a cron job

To run the backup script every 5 minutes:  
Edit your crontab:
```bash
crontab -e
```
- **crontab -e**: Edit your user’s list of scheduled tasks.

Add this line at the end:

```cron
*/5 * * * * ~/db_backups/auto_backup.sh
```
- This schedules your script to run every 5 minutes.

**Summary:**  
- A backup is taken every 5 minutes.  
- Only the 10 most recent backups are kept in `~/db_backups`.  
- Your cron job manages everything automatically.

**Note:** Cron jobs run as your user, so ensure your user can run Docker and access the backup directory.

** Save the crontab file using CNTRL+X
