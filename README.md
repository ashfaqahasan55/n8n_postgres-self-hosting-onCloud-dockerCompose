# Self-Hosting SSL enabled N8N on a Linux Server with docker compose

This guide provides step-by-step instructions to self-host [n8n](https://n8n.io), a free and open-source workflow automation tool, on a Linux server using Docker compose, Nginx, and Certbot for SSL with a custom domain name.



## Step 1: Installing Docker and Docker compose

1. **Set up Docker’s apt repository:**
```bash
    # Add dockers official GPG key:
    sudo apt-get update
    sudo apt-get install ca-certificates curl gnupg
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg

    # Add the repository to Apt sources:
    echo \
    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update 
 ```

2. **Install the latest version:**
    ```bash
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```

3. **check the docker version:**
    ```bash
    docker -v
    ```
    If that returns a version, you’re good to go.

4. **check the docker version:**
    ```bash
    sudo docker compose
    ```
    If that returns al the commands help, you’re good to go.
    


## Step 2: Setting up

1. **Create the Project Folder:**

    I like to keep things organized under /opt/stacks, but you can adjust this to fit your structure:

```bash
    sudo mkdir -p /opt/stacks/n8n
    sudo chown "username":"username" -R /opt/stacks
    cd /opt/stacks/n8n
```
    Replace "username" with your instance username. In my case it's "ubuntu".


2. **Create the Docker Compose File**

```bash
    sudo nano compose.yaml
```

    Then paste the following yaml:
```bash
services:
  n8n:
    image: n8nio/n8n:latest
    restart: always
    ports:
    - "5678:5678"
    env_file:
    - .env
    volumes:
    - ./data:/home/node/.n8n
    - ./files:/files
    depends_on:
    - postgres
    # labels:
    # - "traefik.enable=true"
    # - "traefik.http.routers.${TRAEFIK_ROUTER_NAME}.rule=Host(`${TRAEFIK_DOMAIN}`)"
    # - "traefik.http.routers.${TRAEFIK_ROUTER_NAME}.entrypoints=websecure"
    # - "traefik.http.routers.${TRAEFIK_ROUTER_NAME}.tls.certresolver=${TRAEFIK_CERT_RESOLVER}"
    # - "traefik.http.services.${TRAEFIK_ROUTER_NAME}.loadbalancer.server.port=5678"

# If you're running your own external PostgreSQL instance, you can comment out this service
  postgres:
    image: postgres:15
    restart: always
    env_file:
    - .env
    volumes:
    - ./postgres-data:/var/lib/postgresql/data
        
```

     This gives us a working setup using Docker Compose, with support for environment variables, volumes for persistence, and optional Traefik labels if you want to enable TLS down the line.


3. **Create .env file:**

```bash
    sudo nano .env
```
    Paste in your values:
```bash
# n8n Settings
DOMAIN_NAME=peachsoft.co.uk #replace with your-domain name or server public ip
SUBDOMAIN=optiforgeai #replace it with your subdomain if you have, if not just ease this variable
GENERIC_TIMEZONE=Europe/London
N8N_HOST=optiforgeai.peachsoft.co.uk  #subdomain.your-domain.com
N8N_PROTOCOL=https. #if not hosting globally erase it out
WEBHOOK_URL=https://optiforgeai.peachsoft.co.uk #url for subdomain.your-domain.com (its webhook url for tools to connect globally with other services)
N8N_SECURE_COOKIE=false
N8N_PORT=5678
N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
N8N_RUNNERS_ENABLED=true
NODE_ENV=production

# n8n Database Settings (for the n8n application itself)
DB_POSTGRES_HOST=postgres
DB_POSTGRES_PORT=5432
DB_POSTGRES_DATABASE=postgres # You can choose your n8n database name
DB_POSTGRES_USER=postgres # You can choose your n8n database user
DB_POSTGRES_PASSWORD=ubuntu #change it with a strong pass

# PostgreSQL Container Settings (for the postgres service)
# These are used by the 'postgres' service's entrypoint to configure the database
POSTGRES_DB=${DB_POSTGRES_DATABASE}
POSTGRES_USER=${DB_POSTGRES_USER}
POSTGRES_PASSWORD=${DB_POSTGRES_PASSWORD} # CRITICAL: Must match the n8n DB password

# Optional: Traefik (if you plan to use it for reverse proxying directly via Docker)
# TRAEFIK_ROUTER_NAME=optiforgeai
# TRAEFIK_DOMAIN=${SUBDOMAIN_NAME}.${DOMAIN_NAME}
# TRAEFIK_CERT_RESOLVER=mytlschallenge
```

If not postgresdb not working, try to remove the postgres volume folder to start from the begining.
```bash
sudo rm -rf postgres-data
mkdir postgres-data
```


    

4. **Starting the stack**
Navigate to your project directory:
```bash
  cd /opt/stacks/n8n
```

make folder /data:
```bash
sudo mkdir /opt/stacks/n8n/data
```
make folder /files:
```bash
sudo mkdir /opt/stacks/n8n/files
```

Then bring everything up in the background:
```bash
sudo docker compose up -d

```
5. **Confirm It’s Running**
To confirm everything started correctly, run:
```bash
  sudo docker ps
```
You should see both the n8n and postgres containers listed and running. If not, check the logs with:
```bash
sudo docker compose logs
```
At this point your n8n is accessible locally via port 5678. You will get all the logs and at the end of the logs you can see something like this:
n8n-1       | Editor is now accessible via:
n8n-1       | https://n8n.retaliate.co.uk
n8n-1       | Registered runner "JS Task Runner" (wYVjeHFj5qPSRT6h7BmkF)

The n8n is now accessible via {your domain name or ip address} with port 5678.
In may case it's n8n.retaliate.co.uk:5678
Hold on! still not globally hosted with ssl link. Take a break. Drink coffee, tea. Then proceed to the next section.



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
    server_name our-domain.com; #change it with your submain-domain name(in my case: n8n.retaliate.co.uk)

    location / {
        proxy_pass http://localhost:5678; # Or http://127.0.0.1:5678
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Essential for WebSockets
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400s; # Long timeout for WebSockets

         }
    }
    ```
    Replace your-domain.com with your actual domain.

3. **Enable the Configuration:**
    ```bash
    sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/

    If you see the error saying /etc/nginx/sites-enabled/ doesn't exist. Create it by running: sudo mkdir /etc/nginx/sites-enabled/

4. **Test the Nginx Configuration, Delete the default, and Restart:**
    ```bash
    sudo nginx -t
    sudo rm /etc/nginx/sites-enabled/default
    sudo systemctl restart nginx
    ```

5. **Some cloud services e.g. oracle cloud blocks traffice using IPTABLES. To fix it:**
    for TCP 80:
    ```bash
   sudo iptables -I INPUT 4 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
    ```
    for TCP 443
    ```bash
   sudo iptables -I INPUT 5 -p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
    ```
    then save:
    ```bash
   sudo netfilter-persistent save
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
![image](https://github.com/user-attachments/assets/344187ec-5bcf-4d97-ad35-21b6562182e5)
 

## How to update n8n:

You can follow the instructions here to update the version: https://docs.n8n.io/hosting/installation/docker/#updating

Important: Take a backup of ~/.n8n:/home/node/.n8n
To create a backup, you can copy ~/.n8n:/home/node/.n8n to your local or another directory on the same VM even before deleting the container. And then, after updating and spinning up a new container, if you see the data getting lost, you can replace ~/.n8n:/home/node/.n8n with the one you saved earlier.

Ensure that your n8n instance is using a persistent volume or a mapped directory for its data storage. This is crucial because the workflows, user accounts, and configurations are stored in the database file (typically database.sqlite), which should be located in a directory that remains intact even when the container is removed.
In your docker-compose.yml, you should have something like this:
```bash
volumes:
- /data:/home/node/.n8n
```

This mapping ensures that the .n8n directory on your host machine is used for data storage, preserving your workflows and configurations across container updates.

When you stop and remove the n8n container, you are only deleting the container instance itself, not the data stored in the persistent volume. As long as the volume is correctly configured, your workflows and accounts should remain unaffected.

But to avoid any chance of data loss you should take a backup of ~/.n8n:/home/node/.n8n before removing the container.

## Important Notes
- Ensure your domain's DNS A record points to your server's IP address.
- Allow ports 80 (HTTP), 443 (HTTPS), and 5678 (n8n) in your server's firewall.
- Nginx handles SSL termination, so it forwards requests to the n8n instance over HTTP internally.

## Why Nginx and Certbot?

**Nginx:** It serves as a reverse proxy, forwarding client requests to n8n running on Docker. This setup enhances security, load balancing, and scalability.

**Certbot:** Certbot is a tool from the Electronic Frontier Foundation (EFF) that automates the process of obtaining and renewing SSL certificates from Let's Encrypt, a free and open Certificate Authority.

By using Nginx and Certbot, you ensure that your n8n instance is securely accessible over the internet with HTTPS.

## Troubleshooting

If you encounter issues with Nginx, check the logs located at /var/log/nginx/error.log for more details.
For Docker-related issues, ensure the Docker service is running: sudo systemctl status docker.
