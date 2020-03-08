# NGINX Reverse Proxy

## Environment Variables

Change these to your 'download' and 'plex' web host names.

```shell script
DOWNLOAD_VHOST=download.example.com
PLEX_VHOST=plex.example.com
USER_NAME=damian
```

## Installation

### Install NGINX Container

Create the user, group and runtime directories. Unfortunately the user and group hard coded to a uid/gid of 101 in the official container.

```shell script
sudo groupadd -r -g 101 nginx
sudo useradd -r -u 101 -g nginx -d /var/cache/nginx -s /sbin/nologin -M nginx
sudo mkdir /var/cache/nginx
sudo chown nginx:nginx /var/cache/nginx
echo "d /run/nginx 755 root root" | sudo tee /etc/tmpfiles.d/nginx.conf
sudo systemd-tmpfiles --create
```

Copy over the configuration.

```shell script
sudo mkdir -p /etc/nginx/certs
sudo mkdir -p -m 0700 /etc/nginx/private
sudo cp config/*.conf /etc/nginx/
sudo openssl dhparam -out /etc/nginx/certs/dhparam.pem 2048
# You'll need to install htpasswd, run it on another machine, or generate it another way.
htpasswd -c /etc/nginx/htpasswd $USER_NAME
```

As this service is running on the host network, we need to open the firewall for it:

```shell script
sudo firewall-cmd --add-service http
sudo firewall-cmd --add-service https
sudo firewall-cmd --runtime-to-permanent
```

I like to create a dummy default virtual host with a dummy cert so the actual hostname is needed before
seeing my content. Privacy/security through obscurity, I'm sure it's easy to to bypass.

```shell script
sudo openssl req -newkey rsa:2048 -nodes -x509 -sha256 -days 3650 \
    -subj "/CN=localhost.localdomain" \
    -addext "subjectAltName = DNS:localhost.localdomain" \
    -keyout /etc/nginx/private/localhost.key \
    -out /etc/nginx/certs/localhost.cert
```

This will help you generate a self signed cert, but I recommend https://letsencrypt.org/.

```shell script
for CERT in $DOWNLOAD_VHOST $PLEX_VHOST; do
    sudo openssl req -newkey rsa:2048 -nodes -x509 -sha256 -days 365 \
        -subj "/C=AU/ST=Victoria/CN=${CERT}" \
        -addext "basicConstraints=critical,CA:FALSE" \
        -addext "subjectAltName = DNS:${CERT}" \
        -addext "certificatePolicies = serverAuth,clientAuth" \
        -keyout /etc/nginx/private/${CERT}.key \
        -out /etc/nginx/certs/${CERT}.cert
done
```

Start the official NGINX image as read-only:

```shell script
sudo podman run -d --read-only \
    --name nginx \
    --network host \
    -v /etc/nginx:/etc/nginx:ro,Z \
    -v /var/cache/nginx:/var/cache/nginx:Z \
    -v /run/nginx:/var/run:Z \
    -v /etc/letsencrypt:/etc/letsencrypt:z \
    -p 443:443 \
    nginx
```

Add the container to systemd for service management:

```shell script
sudo cp systemd/nginx-container.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable nginx-container.service
sudo systemctl start nginx-container.service
```

