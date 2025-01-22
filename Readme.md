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


### How to Check Certificate Expiration

You can check the expiration date of your SSL certificates in several ways:

#### Option 1: Using Certbot
Run the following command to list the certificates managed by Certbot and their expiration dates:

```bash
docker-compose run --rm certbot certificates
```

This will output something like:

```
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Found the following certs:
  Certificate Name: abm-app.ddns.net
    Domains: abm-app.ddns.net
    Expiry Date: 2025-04-15 10:22:01+00:00 (VALID: 89 days)
    Certificate Path: /etc/letsencrypt/live/abm-app.ddns.net/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/abm-app.ddns.net/privkey.pem
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

The `Expiry Date` shows the expiration date of your certificate.

#### Option 2: Check the Certificate File
Inspect the certificate directly using `openssl`:

1. Access the certificate path in your container:
   ```bash
   docker-compose exec nginx cat /etc/letsencrypt/live/abm-app.ddns.net/fullchain.pem | openssl x509 -noout -dates
   ```

2. This will output:
   ```
   notBefore=Jan 16 12:00:00 2025 GMT
   notAfter=Apr 15 12:00:00 2025 GMT
   ```

The `notAfter` date is the expiration date.

#### Option 3: Check Online
You can use online tools like [SSL Labs](https://www.ssllabs.com/ssltest/) or [SSL Shopper](https://www.sslshopper.com/) to test your domain and see the certificate's expiration date.

#### Option 4: Automate with a Script
Create a script to check the certificate's expiration:

```bash
docker-compose run --rm certbot certificates | grep 'Expiry Date'
```

You can set up a CRON job to check and send an alert when the expiration date approaches.

#### Renewing the Certificate
Certbot automatically renews certificates 30 days before expiration. To test this, run:

```bash
docker-compose run --rm certbot renew --dry-run
```

This ensures that the renewal process works correctly.

---

Make sure to regularly verify your certificate's expiration and automate the renewal process.


## Conclusion

Your application is now secured with HTTPS. Regularly verify that your certificates are valid and monitor logs for any potential issues.

