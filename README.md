# Wordpress for Docker Swarm with Nginx and Phpmyadmin

### Deploy WordPress with nginx in a Docker Swarm cluster. The setup involves the following components:

- <u>**MySQL**</u>  Database server used for storing WordPress data.
- <u>**WordPress**</u>  The content management system deployed on the cluster.
- <u>**Nginx**</u>  A web server used to serve the WordPress application.
- <u>**Phpmyadmin**</u>  A web interface for managing MySQL databases.

### External Dependencies:
- <u>**Traefik**</u>  A modern reverse proxy and load balancer designed for dynamic container environments, used to route external traffic to the Docker Swarm services.
- <u>**(Optional, recommended) [GlusterFS](https://www.gluster.org/)**</u>  A scalable distributed file system providing high availability for shared storage across the swarm nodes.
- <u>**(Optional) [Swarm Autoscaler](https://github.com/Sokrates1989/swarm-monitoring-autoscaler)**</u>  Automatically scales the Nginx service based on CPU and memory usage metrics to handle varying traffic demands.

### Original Author
This project is based on the original work by [Iiriix](https://github.com/iiriix). Source project: [docker-swarm-wordpress](https://github.com/iiriix/docker-swarm-wordpress). For more info view [Attribution](#attribution).

## Table of Contents
1. [Initial Setup](#initial-setup)
   - [Domains and Subdomains](#domains-and-subdomains)
   - [Setup Repository](#setup-repository)
   - [Create Secrets in Docker Swarm](#create-secrets-in-docker-swarm)
   - [Copy Templates](#copy-templates)
   - [Edit Configuration](#edit-configuration)
     - [.env](#env)
     - [config-stack.yml](#config-stackyml)
2. [Deployment](#deployment)
   - [Determine Readiness](#determine-readiness)
3. [Maintenance](#maintenance)
   - [Purge Cache](#purge-cache)
   - [Scaling](#scaling)
   - [MySQL Database](#mysql-database)
   - [Caching](#caching)
   - [PHP Configuration (php.ini)](#php-phpini)
   - [Volumes](#volumes)
4. [Re-Deploy to Fix Errors](#re-deploy-to-fix-errors)
   - [Backup Database](#backup-database)
   - [Actual Re-Deployment](#actual-re-deployment)
5. [Access Database](#access-database)
   - [Enable Phpmyadmin via Command Line](#option-1-enable-phpmyadmin-via-command-line)
   - [Enable Phpmyadmin via .env](#option-2-enable-phpmyadmin-via-env)
   - [Confirm Phpmyadmin is Enabled](#confirm-phpmyadmin-enabled)
   - [Log in to Phpmyadmin](#log-in-to-phpmyadmin)
6. [Attribution](#attribution)
7. [License](#license)

## Initial Setup

### Domains and Subdomains

Ensure that the domains and subdomains exist and point to the manager of the swarm.

Example for `test.felicitas-wisdom.de`:
- `test.felicitas-wisdom.de`
- `php.test.felicitas-wisdom.de`

### Setup Repository

Set up the repository at the desired location:

```bash
# Choose location on server (glusterfs when using multiple nodes is recommended).
mkdir -p /gluster_storage/swarm/wordpress/<DOMAINNAME>
cd /gluster_storage/swarm/wordpress/<DOMAINNAME>
git clone https://github.com/Sokrates1989/swarm-wordpress-nginx.git .
```

### Create Secrets in Docker Swarm

Create the necessary secrets in Docker Swarm:

```bash
# Naming conventions for "XXXXXXXXX":
# Replace "XXXXXXXXX" with custom name. "MYSQL_ROOTPW_WORDPRESS_XXXXXXXXX": (e.g. using a website called "foo-bar.com" -> MYSQL_ROOTPW_WORDPRESS_FOO_BAR)

# MYSQL_ROOTPW_WORDPRESS_PLACEHOLDER.
vi secret.txt  # Then insert password (Make sure the password does not contain any backslashes "\") and save the file.
docker secret create MYSQL_ROOTPW_WORDPRESS_XXXXXXXXX secret.txt # Change MYSQL_ROOTPW_WORDPRESS_XXXXXXXXX
rm secret.txt

# MYSQL_USERPW_WORDPRESS_PLACEHOLDER.
vi secret.txt  # Then insert password (Make sure the password does not contain any backslashes "\") and save the file.
docker secret create MYSQL_USERPW_WORDPRESS_XXXXXXXXX secret.txt # Change MYSQL_USERPW_WORDPRESS_XXXXXXXXX
rm secret.txt
```

### Copy Templates

Copy the necessary template files:

```bash
# Copy ".env.template" to ".env".
cp .env.template .env

# Copy "config-stack.yml.template" to "config-stack.yml".
cp config-stack.yml.template config-stack.yml
```

### Edit Configuration

#### .env

Edit the variables in the `.env` file:

```bash
vi .env
```

#### config-stack.yml

Replace the placeholder with the previously created [Docker Swarm Secrets](#create-secrets-in-docker-swarm):
```bash
# Change "MYSQL_ROOTPW_WORDPRESS_PLACEHOLDER" to your own secret "MYSQL_ROOTPW_WORDPRESS_XXXXXXXXX".
sed -i -e 's/MYSQL_ROOTPW_WORDPRESS_PLACEHOLDER/MYSQL_ROOTPW_WORDPRESS_XXXXXXXXX/g' ./config-stack.yml

# Change "MYSQL_USERPW_WORDPRESS_PLACEHOLDER" to your own secret "MYSQL_USERPW_WORDPRESS_XXXXXXXXX".
sed -i -e 's/MYSQL_USERPW_WORDPRESS_PLACEHOLDER/MYSQL_USERPW_WORDPRESS_XXXXXXXXX/g' ./config-stack.yml
```

## Deployment

Deploy the service on the swarm using the `.env` file via Docker Compose:

```bash
# https://github.com/moby/moby/issues/29133.
docker stack deploy -c <(docker-compose -f config-stack.yml config)  <STACK_NAME>
```

### Determine Readiness

Check if there are any issues with the initial deployment:

```bash
# Check service status.
docker stack services <STACK_NAME>
# Make sure that the replicas numbers equal left and right side (1/1 and 0/0 is good; 0/1 is bad).

# In case of unequal replicas, check issues of the service.
docker service ps <STACK_NAME>_wordpress --no-trunc
docker service ps <STACK_NAME>_wordpress_db --no-trunc

# Watch until the WordPress logs change to confirm readiness.
watch docker service logs <STACK_NAME>_wordpress
# Look for "Complete! WordPress has been successfully copied to /var/www/html" or "NOTICE: ready to handle connections".

# Desired log entry for WordPress DB.
docker service logs <STACK_NAME>_wordpress_db
# Look for "mysqld: ready for connections".
```

If the above logs do not appear after 20 minutes, call the site via a browser (it should display a 404 error or another error). This will trigger WordPress to start copying files. Continue watching the logs as described above.

## Maintenance

### Purge Cache

Remove the contents of the `nginx_cache` directory to purge the cache:

```bash
rm -rf nginx_cache/*
```

### Scaling

You can scale the `wordpress` and `nginx` services easily:

```bash
# To scale up:
docker service scale my_wordpress_nginx=3
docker service scale my_wordpress_wordpress=3

# Check whether all replicas are running.
docker service ls

# To scale down:
docker service scale my_wordpress_nginx=1
docker service scale my_wordpress_wordpress=1
```

### MySQL Database

The stack uses a MySQL Docker image to initialize the required database. If you prefer not to use Docker for your database, you can remove the MySQL service from the `docker-stack.yml` file and set the correct environment variables for the WordPress database.

### Caching

The Nginx image is configured to support `fastcgi_cache`. It caches FPM responses for 2 days. To see content changes, you need to either wait for 2 days for the cache to be dropped or [remove the cache manually](#purge-cache). If you don't need caching or want to adjust the cache time, you can edit or remove the relevant configurations from the Nginx config file (`./apps/nginx/nginx_conf/conf.d/default.conf`).

### PHP (php.ini)

The `php.ini` file is mounted to the WordPress containers. To change parameters such as `post_max_size` or `upload_max_filesize`, edit the file in `./apps/wordpress/php.ini`.

### Volumes

This stack has three volumes to persist the data:

* `./wordpress_data`: Stores all WordPress files, themes, addons, etc.
* `./db_data`: Stores the database files.
* `./nginx_cache`: Stores cached Nginx sites, allowing you to [purge the cache manually](#purge-cache).

## Re-Deploy to Fix Errors

### Backup Database

Backup the database data using Phpmyadmin:

1. Log into Phpmyadmin.
2. Choose the database in the left panel with the name mentioned in `.env` under `MYSQL_DATABASE_NAME`.
3. Choose "Export" in the top menu.
4. Export the database as SQL.
5. Ensure the file is downloaded.
6. Disable Phpmyadmin again by setting the replicas to 0 as described in the "Access Database" section.

### Actual Re-Deployment

```bash
# Remove all services in the stack.
docker stack rm <STACK_NAME>

# Only if you want to completely restart all data fresh.
# !!! COMPLETE RESTART (old data is moved) !!!
mv wordpress_data/ wordpress_data_old
mv db_data/ db_data_old
git restore db_data/.gitkeep wordpress_data/.gitkeep

# Re-deploy.
docker stack deploy -c <(docker-compose config) <STACK_NAME>
```
## Access Database

### Option 1: Enable Phpmyadmin via Command Line

```bash
# Update service replica.
docker service update --replicas 1 <STACK_NAME>_phpmyadmin
```

### Option 2: Enable Phpmyadmin via .env

```bash
# Go to the repository directory.
cd /gluster_storage/swarm/wordpress/<DOMAINNAME>

# Edit .env.
vi .env
# Set PHPMYADMIN_REPLICAS to 1.

# Re-deploy.
docker stack deploy -c <(docker-compose config) <STACK_NAME>
```

### Confirm Phpmyadmin is Enabled

```bash
docker stack services <STACK_NAME>
```

### Log in to Phpmyadmin

Access the Phpmyadmin interface using the URL set up in `.env`:

- `https://php.test.felicitas-wisdom.de`


## Attribution
This project is based on the original work by [Iiriix](https://github.com/iiriix). Source project: [docker-swarm-wordpress](https://github.com/iiriix/docker-swarm-wordpress). 
Modifications have been made to customize the deployment process, including almost everything, except for the nginx server.

## License
This project is licensed under the terms of the GNU General Public License v3.0. See the [LICENSE](./LICENSE) file for details.
