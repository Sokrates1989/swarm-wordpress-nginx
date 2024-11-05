# Use a specific Alpine version for stability and predictability in builds.
FROM alpine:3.20.3

# Metadata labels for documentation and easy identification.
LABEL maintainer="@Sokrates1989" \
      org.label-schema.description="Runs Nginx in front of a WordPress service in a Docker Swarm cluster" \
      org.label-schema.name="alpine-nginx-cache" \
      org.label-schema.url="https://github.com/Sokrates1989/swarm-wordpress-nginx" \
      org.label-schema.vcs-url="https://github.com/Sokrates1989/swarm-wordpress-nginx" \
      org.label-schema.schema-version="1.0"

# Set up environment and install necessary packages.
RUN set -e && \
    \
    # Install shadow package to create Nginx user and group.
    apk add --no-cache --virtual .build-deps shadow && \
    \
    # Create Nginx user and group with specific UID and GID to match WordPress container.
    # This ensures file permission consistency across containers.
    # Check if GID 82 is available, otherwise find an unused GID for the nginx group.
    if ! getent group nginx >/dev/null 2>&1; then \
        if getent group 82 >/dev/null 2>&1; then \
            # Find an unused GID starting from 1000 if 82 is taken.
            UNUSED_GID=$(awk -F: '{ if ($3 >= 1000) gids[$3]=1 } END { for (i=1000; i<65536; i++) if (!(i in gids)) {print i; exit} }' /etc/group); \
            groupadd -g "$UNUSED_GID" nginx; \
        else \
            groupadd -g 82 nginx; \
        fi; \
    fi && \
    \
    # Check if the nginx user already exists, and if not, create it with the nginx group.
    if ! id -u nginx >/dev/null 2>&1; then \
        useradd -u 82 -g nginx -c "nginx user" -m -d /var/lib/nginx -s /sbin/nologin nginx; \
    fi && \
    \
    # Install Nginx with cache purge module, which allows cached content to be purged.
    apk add --no-cache nginx nginx-mod-http-cache-purge && \
    \
    # Clean up: Remove shadow package to reduce image size.
    apk del .build-deps && \
    \
    # Remove default web root to avoid any potential conflicts with custom configurations.
    rm -rf /var/www/localhost/ && \
    \
    # Create cache directory for fastcgi_cache and assign ownership to Nginx user.
    mkdir -p /var/cache/nginx && \
    chown -R nginx:nginx /var/cache/nginx && \
    \
    # Create PID directory for Nginx.
    mkdir -p /var/run/nginx && \
    \
    # Redirect access and error logs to Docker's log collector.
    ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log


# Expose HTTP port 80.
EXPOSE 80

# Define the signal to stop Nginx gracefully.
STOPSIGNAL SIGTERM

# Start Nginx in the foreground to keep the container running.
CMD ["nginx", "-g", "daemon off;"]
