When you install PGSQL, it will be best to install PGADMIN to manage and view DB. 

PG ADMIN will be installed on linux server but will be accessed through the ip address of the server as follows <up_address/pgadmin4>.

To install pgAdmin on an Ubuntu Linux server for web-based database management, you need to add the official pgAdmin repository and install the
pgadmin4-web package. 
Prerequisites

    An Ubuntu server (versions 20.04, 22.04, or 24.04 recommended).
    A non-root user with sudo privileges.
    PostgreSQL installed and running on the server (or a remote server you can access).
    The curl package installed: sudo apt install curl -y. 

## Installation Steps

Update your system packages:
```
sudo apt update && sudo apt upgrade -y
```

### Add the pgAdmin 4 repository key:
```
curl -fsSL https://www.pgadmin.org/static/packages_pgadmin_org.pub | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/pgadmin.gpg
```

### Add the pgAdmin 4 repository to your sources list:
```
sudo sh -c 'echo "deb https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/$(lsb_release -cs) pgadmin4 main" > /etc/apt/sources.list.d/pgadmin4.list'
```

### Update your package lists again to recognize the new repository:
```
sudo apt update
```


### Install the web version of pgAdmin 4 (this will also install Apache as a web server):
```
sudo apt install pgadmin4-web -y
```

### Run the web setup script to configure the environment and create an initial admin user account:
```
sudo /usr/pgadmin4/bin/setup-web.sh
```

During this script, you will be prompted to provide an email address and password for your pgAdmin login. 

### Access and Configuration

Access the web interface: Open your web browser and navigate to http://YOUR_SERVER_IP_ADDRESS/pgadmin4.

Log in: Enter the email address and password you created during the setup script.

Connect to your PostgreSQL server: 
    
    1. In the pgAdmin dashboard, click Add New Server.
    2. In the General tab, enter a name for your server (e.g., "My Local Server").
    3. Go to the Connection tab.
    4. For Host name/address, enter localhost (or the remote IP/hostname of your PostgreSQL server).
    5. The Port is typically 5432 by default.
    6. Enter the Username (commonly postgres) and the corresponding Password you set for your PostgreSQL user.
    7. Click Save. 

You can now manage your PostgreSQL databases using the pgAdmin web interface. 


Please NOTE that you cannot have any other applications being served to server main domain before you start the webscript setup as this will prevent the Apache server from starting up. 


### ONCE DONE WITH PGADMIN CHANGES:

- You must shitdown the apache server so that you can run the main application on port 80

```
sudo apachectl stop
```

