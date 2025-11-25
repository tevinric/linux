Ensure you have Docker installed on your Ubuntu/Linux machine.

### Step 1: Pull the PostgreSQL Image
Open your terminal and download the official PostgreSQL image from Docker Hub.
bash

    docker pull postgres

### Step 2: Create a Docker Volume for Data Persistence
Create a named volume to ensure your data is stored safely on the host machine and is not lost when the container stops or is removed. This volume will physically reside within the Docker managed area on your Linux server.
bash

        docker volume create pg_data

##### Volume Details and Location:
To find the exact location of this volume on your Linux filesystem, use the following command:

        docker volume inspect pg_data

Look for the "Mountpoint" key in the output (e.g., /var/lib/docker/volumes/pg_data/_data). This directory on your Linux server stores all database files, including user definitions, database schemas, and all inserted data.

### Step 3: Run the PostgreSQL Container and Initialize the Database/User

Run the container using a single command that defines the new user (myuser) and database (mydatabase) via environment variables (-e). The PostgreSQL image is designed to read these variables upon its very first boot and create these resources automatically before the service is ready to accept connections. The -v flag mounts the volume to the necessary directory within the container.

        docker run -d --name my_postgres_container \
          -e POSTGRES_USER=myuser \
          -e POSTGRES_PASSWORD=mysecretpassword \
          -e POSTGRES_DB=mydatabase \
          -v pg_data:/var/lib/postgresql/data \
          -p 5432:5432 \
          postgres




