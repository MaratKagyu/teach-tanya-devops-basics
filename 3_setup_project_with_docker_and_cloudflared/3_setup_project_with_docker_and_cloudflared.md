This is just an educational TODO. Please do not consider it as a reliable solution to run a project on prod.
But it's still pretty useful to run a pet project, especially if you don't have a dedicated server exposed out to 
the internet.


# 1. Install Docker (if it's not installed yet) on Debian
To see how to install Docker on another OS see: https://docs.docker.com/engine/install/ 

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install Docker
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

# 2. Prepare the docker-compose file tuned for production
Recommendations:
- Do not expose any ports of the containers
- Get rid of dev-related dependencies if you have some

Here's the example configuration for `nginx` + `php-fpm` + `psql` tuned for the Yii2 framework.

`./docker-compose.prod.yml`:
```yaml
services:
  fpm:
    build:
      context: .docker/local/php-fpm
      dockerfile: Dockerfile
    volumes:
      - ./:/var/www/html
    networks:
      - project-dev

  nginx:
    build:
      context: .docker/local/nginx
      dockerfile: Dockerfile
    volumes:
      - ./:/var/www/html
    networks:
      - project-dev
    depends_on:
      fpm:
        condition: service_started

  database:
    image: postgres:15-alpine
    env_file: ./.env
    networks:
      - project-dev
    volumes:
      - database_data:/var/lib/postgresql/data:rw

networks:
  project-dev:
    driver: bridge

volumes:
  database_data:
```

The other related files/configs can be found in `./3_setup_project_with_docker_and_cloudflared/` of this repo.

Make sure `./env` file contains these variables `POSTGRES_DB`, `POSTGRES_PASSWORD`, `POSTGRES_USER`.

# 3. Test run

```bash
# Run the cluster
sudo docker compose -f ./docker-compose.prod.yml up 
```
Provide so checks, like:
- Run commands
- Try to wget the nginx container

If everything is fine, you can turn is down (Press Ctrl+C). And drop the containers:
```dockerfile

sudo docker compose -f ./docker-compose.prod.yml down
```

# 4. Setup a Cloudflare tunnel
1. You must have a CloudFlare account with at least one domain setup. 
2. Login into your CloudFlare account
3. Click on ZeroTrust. Pass all necessary steps, if you didn't set it up previously (Select a Team Name, Choose a Plan
a free one must work for you)
4. Click on Networks → Tunnels → Add Tunnel
5. "Select Cloudflared"
6. Name the tunnel, press Save
7. Copy the token (any example command will have it), press Next
8. Select your domain and fill in a subdomain
9. Select Service Type: HTTP
10. URL: "nginx"
11. Press "Complete Setup"
12. Wait a bit and open the site (your subdomain + domain) in the Browser. You'll find Error 1033.

# 5. Set up a `cloudflared` container inside docker-compose
1. Add a new container to `docker-compose.prod.yml`:
```yml
  cloudflared:
    image: cloudflare/cloudflared:latest
    env_file: ./.env
    command: tunnel --no-autoupdate run --token ${CF_TUNNEL_TOKEN}
#    environment:
#      - TUNNEL_METRICS=0.0.0.0:49999
    networks: [project-dev]
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "cloudflared", "version"]
      interval: 30s
      timeout: 5s
      retries: 3
```

2. Add `CF_TUNNEL_TOKEN` to your `.env` file with the token you copied earlier.  
3. Run the cluster:
```bash

sudo docker compose -f ./docker-compose.prod.yml up
```
4. Now the site must become available. 
5. Stop the cluster (Ctrl+C) and make sure it's `down`.

# 6. Set up the cluster as system service
Replace `/var/www/slots-bot` with the project location.
Replace `docker-compose-slots-bot` with whatever name you want

1. Setup service config
```bash

sudo nano /etc/systemd/system/docker-compose-slots-bot.service
```
Paste this:
```
[Unit]
Description=Slots web app
After=network-online.target docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=true
WorkingDirectory=/var/www/slots-bot
ExecStart=/usr/bin/docker compose -f /var/www/slots-bot/docker-compose.prod.yml up -d
ExecStop=/usr/bin/docker compose -f /var/www/slots-bot/docker-compose.prod.yml down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```
2. Reload Systemd and Enable the Service
```bash

sudo systemctl daemon-reload
sudo systemctl enable docker-compose-slots-bot.service 
```

3. Start the service
```bash

sudo systemctl start docker-compose-slots-bot.service 
# You can verify the status by running this command
sudo systemctl status docker-compose-slots-bot.service
```

To remove the service, you need to run:
```bash
# Stop the service 
sudo systemctl stop docker-compose-slots-bot.service
# Disable the service
sudo systemctl disable docker-compose-slots-bot.service
# Delete the service file
sudo rm /etc/systemd/system/docker-compose-slots-bot.service
# Reload the systemctl daemon
sudo systemctl daemon-reload
```
