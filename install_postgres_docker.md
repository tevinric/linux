Ensure you have Docker installed on your Ubuntu/Linux machine.

### Step 1: Pull the PostgreSQL Image
Open your terminal and download the official PostgreSQL image from Docker Hub.
bash

    docker pull postgres:16

### Step 2: Create a Docker Volume for Data Persistence
Create a named volume to ensure your data is stored safely on the host machine and is not lost when the container stops or is removed. This volume will physically reside within the Docker managed area on your Linux server.
bash

    docker volume create pg_data_v16

##### Volume Details and Location:
To find the exact location of this volume on your Linux filesystem, use the following command:

    docker volume inspect pg_data

Look for the "Mountpoint" key in the output (e.g., /var/lib/docker/volumes/pg_data/_data). This directory on your Linux server stores all database files, including user definitions, database schemas, and all inserted data.

### Step 3: Run the PostgreSQL Container and Initialize the Database/User

Run the container using a single command that defines the new user (myuser) and database (mydatabase) via environment variables (-e). The PostgreSQL image is designed to read these variables upon its very first boot and create these resources automatically before the service is ready to accept connections. The -v flag mounts the volume to the necessary directory within the container.

    docker run -d --name my_postgres_container_v16 \
      -e POSTGRES_USER=myuser \
      -e POSTGRES_PASSWORD=mysecretpassword \
      -e POSTGRES_DB=mydatabase \
      -v pg_data_v16:/var/lib/postgresql/data \
      -p 5432:5432 \
      postgres:16

Key Explanation: The mapping -v pg_data:/var/lib/postgresql/data ensures that the database's internal data directory is written to your persistent volume on the Linux server.

### Step 4: Verify the Container is Running

Confirm the container started successfully:

    docker ps
    
You should see my_postgres_container with a status of Up....

### Step 5: Access the Database and Create a Table
The user and database now exist. You can connect using the psql command-line client within the running container.

1. Access the shell inside your container as the user you just created to connect to your specific database:

        docker exec -it my_postgres_container_v16 psql -U myuser -d mydatabase

You may be prompted for your password (mysecretpassword).

2. Once you see the mydatabase=> prompt, you are connected. Create a table:

        CREATE TABLE products (
          id SERIAL PRIMARY KEY,
          name VARCHAR(100),
          price NUMERIC(10, 2)
        );

3. Insert some data:

        INSERT INTO products (name, price) VALUES ('Laptop', 999.99), ('Mouse', 25.50);

4. Verify the data:

        SELECT * FROM products;
   
6. Type \q and press Enter to exit the psql shell and return to your Linux terminal.


### Step 6: Backing up Your Data

Since all data is stored in the pg_data volume, you can back it up using standard Linux tools targeting the Mountpoint path found in Step 2. A more reliable method for a live database is using the pg_dump utility.

##### Create a local directory for backups
    mkdir -p ~/db_backups

##### Use pg_dump inside the container to create an SQL dump file on your host machine
    docker exec -t my_postgres_container pg_dump -U myuser mydatabase > ~/db_backups/mydatabase_dump_$(date +\%Y\%m\%d).sql

This ensures all your users, databases, and tables are backed up into a single, restorable SQL file stored safely on your Linux host server.






