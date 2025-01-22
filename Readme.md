# Guide: Configure Nginx with HTTPS for a Docker Application

This guide provides detailed steps to secure your web application running with Nginx and Docker by setting up HTTPS.

## Prerequisites

- A running application on Docker (e.g., Tomcat on port 8080).
- A domain configured with a DDNS provider pointing to your public IP.
- Ports 80 and 443 opened in your firewall and router.
- Docker and Docker Compose installed on your server.

## Steps

### 1. Create the Required Directories

Run the following command to create the necessary directories:

```bash
mkdir -p ./nginx/conf.d ./data/certbot/conf ./data/certbot/www
```

### 2. Set Up Nginx with Tomcat Over HTTP

Configure a `docker-compose.yml` file to include Nginx and Tomcat services.

Example file:

```yaml
version: "3.8"
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./data/certbot/conf:/etc/letsencrypt
      - ./data/certbot/www:/var/www/certbot
    ports:
      - "80:80"
    networks:
      - my-app-net

  app:
    image: tomcat:9
    container_name: tomcat
    ports:
      - "8080:8080"
    networks:
      - my-app-net

networks:
  my-app-net:
    driver: bridge
```

Create a `my-app.conf` file in `./nginx/conf.d/` to configure Nginx as a reverse proxy:

```nginx
server {
    listen 80;
    server_name abm-app.ddns.net;

    location / {
        proxy_pass http://app:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Start the containers using:

```bash
docker-compose up -d
```

### 3. Generate an SSL Certificate with Certbot

Add a `certbot` service to your `docker-compose.yml`:

```yaml
  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - ./data/certbot/conf:/etc/letsencrypt
      - ./data/certbot/www:/var/www/certbot
    networks:
      - my-app-net
```

Generate the SSL certificate using the following command:

```bash
docker-compose run --rm certbot certonly --webroot -w /var/www/certbot -d abm-app.ddns.net
```

### 4. Configure HTTPS in Nginx

Update `my-app.conf` to include HTTPS configuration:

```nginx
server {
    listen 80;
    server_name abm-app.ddns.net www.abm-app.ddns.net;
    return 301 https://abm-app.ddns.net$request_uri;
}

server {
    listen 443 ssl;
    server_name abm-app.ddns.net;

    ssl_certificate /etc/letsencrypt/live/abm-app.ddns.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/abm-app.ddns.net/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        proxy_pass http://app:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Reload Nginx to apply the changes:

```bash
docker-compose exec nginx nginx -s reload
```

### 5. Automatically Renew Certificates

Certbot allows for automatic certificate renewal. Use the following command to renew the certificates:

```bash
docker-compose run --rm certbot renew
```

Add a CRON job to run this command regularly and reload Nginx:

```bash
0 0 * * 0 docker-compose run --rm certbot renew && docker-compose exec nginx nginx -s reload
```

## Conclusion

Your application is now secured with HTTPS. Regularly verify that your certificates are valid and monitor logs for any potential issues.

