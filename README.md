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
     - [config-stack.yml](#config-stackyml)
       - [Changing the WordPress Image](#changing-the-wordpress-image)
     - [Cache Configuration](#cache-configuration)
        - [Replacement Commands for Cache Configuration](#replacement-commands-for-cache-configuration)
        - [Suggested Configurations Based on Traffic Scenarios](#suggested-configurations-based-on-traffic-scenarios)
     - [.env Configuration](#env)

2. [Deployment](#deployment)
   - [Determine Readiness](#determine-readiness)
   - [Install Wordpress](#install-wordpress)
   - [Clear Server Cache](#clear-server-cache)

3. [Maintenance](#maintenance)
   - [Purge Cache](#purge-cache)
   - [Scaling](#scaling)
     - [Manual Scaling](#manual-scaling)
     - [Automatic Scaling](#automatic-scaling)
   - [Database Management](#mysql-database)
   - [Caching in Nginx](#caching)
   - [PHP Configuration](#php-phpini)
   - [Volumes](#volumes)

4. [Re-Deploy to Fix Errors](#re-deploy-to-fix-errors)
   - [Backup Database](#backup-database)
   - [Actual Re-Deployment](#actual-re-deployment)
   - [Wait for Readiness](#wait-for-readiness)

5. [Access Database via Phpmyadmin](#access-database)
   - [Enable Phpmyadmin via Command Line](#option-1-enable-phpmyadmin-via-command-line)
   - [Enable Phpmyadmin via .env](#option-2-enable-phpmyadmin-via-env)
   - [Log in to Phpmyadmin](#log-in-to-phpmyadmin)

6. [Developer/Maintainer Information](#developermaintainer-information)
   - [Building and Pushing the Alpine Nginx Wordpress Docker Image](#building-and-pushing-the-alpine-nginx-wordpress-docker-image)

7. [Attribution](#attribution)

8. [License](#license)


# Initial Setup

## Domains and Subdomains

Ensure that the domains and subdomains exist and point to the manager of the swarm.

Example for `test.felicitas-wisdom.de`:
- `test.felicitas-wisdom.de`
- `db.test.felicitas-wisdom.de`

## Setup Repository

Set up the repository at the desired location:

```bash
# Choose location on server (glusterfs when using multiple nodes is recommended).
mkdir -p /gluster_storage/swarm/wordpress/<DOMAINNAME>
cd /gluster_storage/swarm/wordpress/<DOMAINNAME>
git clone https://github.com/Sokrates1989/swarm-wordpress-nginx.git .
```

## Create Secrets in Docker Swarm

Create the necessary secrets in Docker Swarm:

```bash
# Naming conventions for "XXXXXXXXX":
# Replace "XXXXXXXXX" with custom name. "MYSQL_ROOTPW_WORDPRESS_XXXXXXXXX": (e.g. using a website called "foo-bar.com" -> MYSQL_ROOTPW_WORDPRESS_FOO_BAR)

# MYSQL_ROOTPW_WORDPRESS_PLACEHOLDER.
vi secret.txt  # Then insert password (Make sure the password does not contain any backslashes (\) single quotes (') or double quotes (") ) and save the file.
docker secret create MYSQL_ROOTPW_WORDPRESS_XXXXXXXXX secret.txt # Change MYSQL_ROOTPW_WORDPRESS_XXXXXXXXX
rm secret.txt

# MYSQL_USERPW_WORDPRESS_PLACEHOLDER.
vi secret.txt  # Then insert password (Make sure the password does not contain any backslashes (\) single quotes (') or double quotes (") ) and save the file.
docker secret create MYSQL_USERPW_WORDPRESS_XXXXXXXXX secret.txt # Change MYSQL_USERPW_WORDPRESS_XXXXXXXXX
rm secret.txt
```

## Copy Templates

Copy the necessary template files:

```bash
# Copy ".env.template" to ".env".
cp .env.template .env

# Copy "config-stack.yml.template" to "config-stack.yml".
cp config-stack.yml.template config-stack.yml

# Copy nginx default.conf template.
cp apps/nginx/nginx_conf/conf.d/default.conf.template apps/nginx/nginx_conf/conf.d/default.conf
```

## Edit Configuration

### config-stack.yml

Replace the placeholder with the previously created [Docker Swarm Secrets](#create-secrets-in-docker-swarm)
```bash
# Replace placeholders with your your own secrets.
sed -i -e 's/MYSQL_ROOTPW_WORDPRESS_PLACEHOLDER/MYSQL_ROOTPW_WORDPRESS_XXXXXXXXX/g' ./config-stack.yml
sed -i -e 's/MYSQL_USERPW_WORDPRESS_PLACEHOLDER/MYSQL_USERPW_WORDPRESS_XXXXXXXXX/g' ./config-stack.yml


# Example for test.felicitas-wisdom.de using Secret names "MYSQL_ROOTPW_WORDPRESS_TEST_FEWI_DE" and "MYSQL_USERPW_WORDPRESS_TEST_FEWI_DE".
sed -i -e 's/MYSQL_ROOTPW_WORDPRESS_PLACEHOLDER/MYSQL_ROOTPW_WORDPRESS_TEST_FEWI_DE/g' ./config-stack.yml
sed -i -e 's/MYSQL_USERPW_WORDPRESS_PLACEHOLDER/MYSQL_USERPW_WORDPRESS_TEST_FEWI_DE/g' ./config-stack.yml
```

### Changing the WordPress Image

In the config-stack.yml file, you can specify which WordPress Docker image you want to use. This allows you to choose different PHP versions or specialized WordPress images that fit your deployment needs.

#### For example, to use WordPress with PHP 8.3 (as provided by the default image)

```yaml
services:
  wordpress:
    image: wordpress:php8.3-fpm-alpine
```

#### If you need to downgrade to PHP 8.2 for compatibility reasons, you can change the image like this

```yaml
services:
  wordpress:
    image: wordpress:6.6-php8.2-fpm-alpine
```

You can find and explore all official WordPress Docker images, including those with different PHP versions and configurations, on the official Docker Hub WordPress page:

[Official WordPress Docker Hub Images](https://hub.docker.com/_/wordpress)

### Cache Configuration

The following settings allow you to control the cache size and duration in Nginx. This can be useful in performance optimization by controlling how long and how much data is cached, reducing server load and speeding up response times.


#### Replacement Commands for Cache Configuration

Use the following commands to replace placeholders in the `default.conf` file with real values for cache size and duration.

```bash
# Replace the cache size placeholder with your desired value.
sed -i -e 's/NGINX_MAX_CACHE_SIZE_PLACEHOLDER/1g/g' ./apps/nginx/nginx_conf/conf.d/default.conf

# Replace the cache duration placeholder with your desired value.
sed -i -e 's/NGINX_CACHE_DURATION_PLACEHOLDER/30m/g' ./apps/nginx/nginx_conf/conf.d/default.conf
```


#### What values should I use?
- **NGINX_MAX_CACHE_SIZE_PLACEHOLDER**: Specifies the maximum size of the cache.
    - **Supported Units**: 
        - k: Kilobytes (1 k = 1024 bytes)
        - m: Megabytes (1 m = 1024 kilobytes)
        - g: Gigabytes (1 g = 1024 megabytes)
    - **Recommended Values**: 
        - Small Site: `500m`
        - Medium Site: `1g`
        - Large Site: `5g` or more based on traffic

- **NGINX_CACHE_DURATION_PLACEHOLDER**: Determines how long cached data is kept before it is considered stale.
    - **Supported Units**: 
        - ms: Milliseconds (1 ms = 0.001 seconds)
        - s: Seconds (1 s = 1000 ms)
        - m: Minutes (1 m = 60 seconds)
        - h: Hours (1 h = 60 minutes)
        - d: Days (1 d = 24 hours)
    - **Recommended Values**: 
        - Low Traffic Sites: `30m` (30 minutes)
        - Medium Traffic Sites: `1h` (1 hour)
        - High Traffic Sites: `6h` (6 hours)

#### Suggested Configurations Based on Traffic Scenarios

1. **Low Traffic Blog or Small Site**: 
    - Cache Duration: `15m`
    - Cache Size: `200m`
    - This setup is ideal for small blogs or websites with limited traffic, providing a balance between caching efficiency and storage usage.

2. **Medium Traffic Business Site**:
    - Cache Duration: `1h`
    - Cache Size: `1g`
    - Suitable for sites with moderate traffic where performance is important, but the cache size can still be limited.

3. **High Traffic E-commerce or News Site**:
    - Cache Duration: `6h`
    - Cache Size: `5g`
    - High-traffic sites can benefit from extended cache durations and larger cache sizes to handle heavy loads and frequent user requests.

### .env

Replace the default domain with the actual domain as setup in [Domains and Subdomains](#domains-and-subdomains) 
```bash
sed -i -e 's/default-domain.com/your-domain.com/g' ./.env

# Example for test.felicitas-wisdom.de.
sed -i -e 's/default-domain.com/test.felicitas-wisdom.de/g' ./.env
```

Edit the variables in the `.env` file:
```bash
vi .env
```

# Deployment

Deploy the service on the swarm using the `.env` file via Docker Compose:

```bash
# https://github.com/moby/moby/issues/29133.
docker stack deploy -c <(docker-compose -f config-stack.yml config) <STACK_NAME>
```

### Determine Readiness

#### Check if there are any issues with the initial deployment
```bash
# Check service status.
docker stack services <STACK_NAME>
# Make sure that the replicas numbers equal left and right side (1/1 and 0/0 is good; 0/1 is bad).

# In case of unequal replicas, check issues of the service.
docker service ps <STACK_NAME>_wordpress --no-trunc
docker service ps <STACK_NAME>_db --no-trunc
```


#### Wait until Wordpress and Database are ready
```bash
# Watch until the WordPress logs change to confirm readiness.
watch docker service logs <STACK_NAME>_wordpress
# Look for "Complete! WordPress has been successfully copied to /var/www/html" or "NOTICE: ready to handle connections".

# Desired log entry for WordPress DB.
docker service logs <STACK_NAME>_db
# Look for "mysqld: ready for connections".
```

If the above logs do not appear after 20 minutes, call the site via a browser (it should display a 404 error or another error). This will trigger WordPress to start copying files. Continue watching the logs as described above.

### Install Wordpress

Go to the url setup in [Domains and Subdomains](#domains-and-subdomains) and follow the short wordpress install guide to install wordpress.

### Clear Server Cache

After having installed wordpress for the first time, nginx still has a redirect to the install page cached, so please [purge the cache](#purge-cache). You will then have successfully installed wordpress on your swarm with a scalable, caching nginx proxy.

# Maintenance

## Purge Cache

Remove the contents of the `nginx_cache` directory to purge the cache:

```bash
rm -rf nginx_cache/*
```

No need to reload anything afterwards -> If ngninx notices, there is no cache anymore, it will automatically call wordpress to create new cache, once someone calls the page.
- However, if you also want that a new cache to be created for certain pages (mostly the homepage) -> Simply open thoses pages in any webbrowser as a non-logged-in user (Open new browser in incognity mode for example).

## Scaling

### Manual Scaling

You can scale the `wordpress` and `nginx` services manually. Since nginx caches the content anyhow I would suggest to keep wordpress at 1 replica and only scale nginx.

```bash
# To scale up:
docker service scale <STACK_NAME>_nginx=3
docker service scale <STACK_NAME>_wordpress=3

# Check whether all replicas are running.
docker service ls

# To scale down:
docker service scale <STACK_NAME>_nginx=1
docker service scale <STACK_NAME>_wordpress=1
```

### Automatic Scaling

By changing the autoscaler settings in .env and installing [Swarm Autoscaler](https://github.com/Sokrates1989/swarm-monitoring-autoscaler), the nginx replicas can be scaled based on the nginx serivce workload.

## MySQL Database

The stack uses a MySQL Docker image to initialize the required database. If you prefer not to use Docker for your database, you can remove the MySQL service from the `docker-stack.yml` file and set the correct environment variables for the WordPress database.

## Caching

The Nginx image is configured to support `fastcgi_cache`. It caches FPM responses for the duration setup in [Cache Configuration](#cache-configuration). To see content changes, you need to either wait for this duration for the cache to be dropped or [remove the cache manually](#purge-cache). If you don't need caching or want to adjust the cache time, you can edit or remove the relevant configurations from the Nginx config file (`./apps/nginx/nginx_conf/conf.d/default.conf`).

## PHP (php.ini)

The `php.ini` file is mounted to the WordPress containers. To change parameters such as `post_max_size` or `upload_max_filesize`, edit the file in `./apps/wordpress/php.ini`.

## Volumes

This stack has three volumes to persist the data:

* `./wordpress_data`: Stores all WordPress files, themes, addons, etc.
* `./db_data`: Stores the database files.
* `./nginx_cache`: Stores cached Nginx sites, allowing you to [purge the cache manually](#purge-cache).

# Re-Deploy to Fix Errors

### Backup Database

Backup the database data using [Phpmyadmin](#access-database):

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

# Only if you want to completely re-install wordpress fresh.
# !!! RE-INSTALL WORDPRESS COMPLETELY (old data is MOVED, not deleted) !!!
mv wordpress_data/ wordpress_data_old
mv db_data/ db_data_old
rm -rf nginx_cache
# Alternative to moving (COMPLETELY ERASES DATA): rm -rf wordpress_data db_data nginx_cache

# Re-Create directories.
git restore db_data/.gitkeep wordpress_data/.gitkeep nginx_cache/.gitkeep

# Re-deploy.
docker stack deploy -c <(docker-compose -f config-stack.yml config) <STACK_NAME>
```

### Wait for readiness
[Determine Readiness](#determine-readiness)

# Access Database

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

- Example: `https://db.test.felicitas-wisdom.de`


# Developer/Maintainer Information

## Building and Pushing the Alpine Nginx Wordpress Docker Image

To build and push the Alpine Nginx Wordpress Docker image with both the `:latest` and `:1.26.2` tags, follow these steps:

### Building the Alpine Nginx Wordpress Docker Image

Build the Docker image locally with the following commands:

```bash
docker build -t sokrates1989/alpine-nginx-wp:latest -f apps/nginx/Dockerfile .
docker build -t sokrates1989/alpine-nginx-wp:1.26.2 -f apps/nginx/Dockerfile .
```

### Pushing to Docker Hub

After building the image, push it to your Docker Hub repository with both tags:

```bash
docker push sokrates1989/alpine-nginx-wp:latest
docker push sokrates1989/alpine-nginx-wp:1.26.2
```

## Attribution
This project is based on the original work by [Iiriix](https://github.com/iiriix). Source project: [docker-swarm-wordpress](https://github.com/iiriix/docker-swarm-wordpress). 
Modifications have been made to customize the deployment process, including almost everything, except for the nginx server.

## License
This project is licensed under the terms of the GNU General Public License v3.0. See the [LICENSE](./LICENSE) file for details.
