# 1. Pull the PostgreSQL 16 image

First, explicitly pull the PostgreSQL version 16 image from Docker Hub to your local machine

    docker pull postgres:16

# 2. Run the container with a new user and database 

Run the container using the docker run command with the required environment variables (-e) to create a new user and database during the initial setup

    docker run --name some-postgres -e POSTGRES_USER=myuser -e POSTGRES_PASSWORD=mypassword -e POSTGRES_DB=mydatabase -p 5432:5432 -d postgres:16

--name some-postgres: Assigns a name to your container for easy reference.
-e POSTGRES_USER=myuser: Sets the desired username. This user will have superuser privileges.
-e POSTGRES_PASSWORD=mypassword: Sets the password for the specified user. This variable is required.
-e POSTGRES_DB=mydatabase: Creates a new database with this name and grants access to the specified user.
-p 5432:5432: Maps the container's PostgreSQL port (5432) to the same port on your host machine.
-d: Runs the container in detached mode (in the background).
postgres:16: Specifies the image name and tag to use. 

# 3. Verify the container is running

    docker ps

# 4. Connect to the database

You can connect to your new database using a PostgreSQL client (like psql if installed locally) or by running the psql client inside the container. 

    docker exec -it some-postgres psql -U myuser -d mydatabase

Once connected, you can execute SQL commands, such as listing the current users:

    \du
