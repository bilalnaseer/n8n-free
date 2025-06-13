# Youtube Tutorial 
[![Install N8N For Free](https://img.youtube.com/vi/x49ZiJDIVPQ/0.jpg)](https://www.youtube.com/watch?v=x49ZiJDIVPQ)


# Self-Hosting SSL enabled N8N on a Linux Server

This guide provides step-by-step instructions to self-host [n8n](https://n8n.io), a free and open-source workflow automation tool, on a Linux server using Docker, Nginx, and Certbot for SSL with a custom domain name.



## Step 1: Installing Docker

1. **Update the Package Index:**
   ```bash
   sudo apt update

2. **Install Docker:**
    ```bash
    sudo apt install docker.io

3.  **Start Docker:**
    ```bash
    sudo systemctl start docker

4. **Enable Docker to Start at Boot:**
    ```bash
    sudo systemctl enable docker


## Step 2: Starting n8n in Docker

**Run the following command to start n8n in Docker. Replace your-domain.com with your actual domain name:**

```bash
    sudo docker run -d --restart unless-stopped -it \
    --name n8n \
    -p 5678:5678 \
    -e N8N_HOST="your-domain.com" \
    -e WEBHOOK_TUNNEL_URL="https://your-domain.com/" \
    -e WEBHOOK_URL="https://your-domain.com/" \
    -v ~/.n8n:/root/.n8n \
    n8nio/n8n
```

**Or if you are using a subdomain, it should look like this:**

```bash
    sudo docker run -d --restart unless-stopped -it \
    --name n8n \
    -p 5678:5678 \
    -e N8N_HOST="subdomain.your-domain.com" \
    -e WEBHOOK_TUNNEL_URL="https://subdomain.your-domain.com/" \
    -e WEBHOOK_URL="https://subdomain.your-domain.com/" \
    -v ~/.n8n:/root/.n8n \
    n8nio/n8n
```


This command does the following:

- Downloads and runs the n8n Docker image.
- Exposes n8n on port 5678.
- Sets environment variables for the n8n host and webhook tunnel URL.
- Mounts the n8n data directory for persistent storage.
- After executing the command, n8n will be accessible on your-domain.com:5678.

## Step 3: Installing Nginx

Nginx is used as a reverse proxy to forward requests to n8n and handle SSL termination.

1. **Install Nginx:**
    ```bash
    sudo apt install nginx

## Step 4: Configuring Nginx

Configure Nginx to reverse proxy the n8n web interface:

1. **Create a New Nginx Configuration File:**
    ```bash
    sudo nano /etc/nginx/sites-available/n8n.conf

2. **Paste the Following Configuration:**
    ```bash
    server {
        listen 80;
        server_name your-domain.com; // subdomain.your-domain.com if you have a subdomain

        location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_read_timeout 86400;
        }
    }
    ```
    Replace your-domain.com with your actual domain.

3. **Enable the Configuration:**
    ```bash
    sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/

4. **Test the Nginx Configuration and Restart:**
    ```bash
    sudo nginx -t
    sudo systemctl restart nginx
    ```

## Step 5: Setting up SSL with Certbot

Certbot will obtain and install an SSL certificate from Let's Encrypt.

1. **Install Certbot and the Nginx Plugin:**
    ```bash
    sudo apt install certbot python3-certbot-nginx

2. **Obtain an SSL Certificate:**
    ```bash
    sudo certbot --nginx -d your-domain.com
    // If you have a subdomain then it will be subdomain.your-domain.com

Follow the on-screen instructions to complete the SSL setup.
Once completed, n8n will be accessible securely over HTTPS at your-domain.com.

IMPORTANT: Make sure you follow the above steps in order. Step 5 will modify your /etc/nginx/sites-available/n8n.conf file to something like this:

![image](/ssl.png)
 


By using Nginx and Certbot, you ensure that your n8n instance is securely accessible over the internet with HTTPS.

