# Server Setup Documentation

Complete guide for setting up a production server with Node.js, Nginx, MongoDB, and PM2.

## Table of Contents

1. [Initial Server Setup](#initial-server-setup)
2. [Firewall Configuration](#firewall-configuration)
3. [Node.js Installation](#nodejs-installation)
4. [GitHub SSH Configuration](#github-ssh-configuration)
5. [Project Deployment](#project-deployment)
6. [Nginx Configuration](#nginx-configuration)
7. [Process Management with PM2](#process-management-with-pm2)
8. [MongoDB Installation](#mongodb-installation)
9. [SSL Certificate Setup](#ssl-certificate-setup)

---

## Initial Server Setup

### Step 1: Connect to Your Server

```bash
ssh root@<ip_address>
```

Enter your password when prompted.

### Step 2: Update System Packages

```bash
sudo apt update
sudo apt upgrade
```

Type `yes` when prompted to confirm the upgrade.

---

## Firewall Configuration

### Enable UFW (Uncomplicated Firewall)

```bash
sudo ufw enable
```

### Allow SSH Access

```bash
sudo ufw allow 22
```

### Check Firewall Status

```bash
sudo ufw status
```

---

## Node.js Installation

### Install NVM (Node Version Manager)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

**Note:** If the version doesn't show, close the terminal and reopen it.

### Install Node.js

```bash
nvm -v
nvm install node
npm -v
node -v
```

### List Installed Versions

```bash
nvm list
```

---

## GitHub SSH Configuration

### Check Existing SSH Keys

```bash
ls -al ~/.ssh
```

### Generate New SSH Key

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

### Start SSH Agent

```bash
eval "$(ssh-agent -s)"
```

### Copy SSH Public Key

```bash
cat ~/.ssh/id_rsa.pub
```

### Add SSH Key to GitHub

1. Go to GitHub → Your Organization → Settings → SSH and GPG keys
2. Click "New SSH key"
3. Add a title (e.g., project name)
4. Paste the SSH key
5. Save

### Test GitHub Connection

```bash
ssh -T git@github.com
```

---

## Project Deployment

### Step 1: Create Project Directory

```bash
cd /var/www
```

If the `www` folder doesn't exist:

```bash
mkdir /var/www
cd /var/www
```

### Step 2: Check DNS Propagation

Visit [https://www.whatsmydns.net/](https://www.whatsmydns.net/) to verify your domain's DNS records.

### Step 3: Clone Project from GitHub

```bash
git clone <github_project_ssh_link>
cd <github-project>
npm i
npm run build
```

---

## Nginx Configuration

### Allow HTTP and HTTPS Traffic

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
```

### Install Nginx

```bash
sudo apt install nginx
nginx -v
sudo systemctl status nginx
```

Press `q` to quit the status view.

### Configure Nginx

```bash
cd /etc/nginx/sites-enabled/
rm default

cd /etc/nginx/sites-available/
rm default
```

### Create New Configuration File

```bash
nano <your_domain_or_ip>
```

**Convention:** Name the file with your domain name or IP address.

### Backend Configuration

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen 80;
    server_name your_domain_or_ip;
    client_max_body_size 100M;

    location / {
        proxy_pass http://localhost:8016;

        # HTTP headers
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # SSE support
        proxy_buffering off;
        proxy_cache off;
        proxy_set_header Cache-Control no-cache;
    }
}
```

### Frontend Configuration

```nginx
server {
    listen 80;
    server_name your_domain_or_ip;
    client_max_body_size 100M;

    location / {
        proxy_pass http://localhost:8016;

        # HTTP headers
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Test and Enable Configuration

```bash
sudo nginx -t
```

Expected output:
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### Create Symbolic Link

```bash
cd /etc/nginx/sites-enabled
ln -s ../sites-available/<your_config_file>
```

### Restart Nginx

```bash
systemctl restart nginx
sudo nginx -s reload
```

---

## Redis Installation (Optional)

```bash
sudo apt install redis-server -y
sudo systemctl enable redis-server
sudo systemctl status redis-server
```

### Test Redis

```bash
redis-cli ping
```

Expected output: `PONG`

### Restart Redis

```bash
sudo systemctl restart redis-server
```

---

## Process Management with PM2

PM2 (Process Manager 2) is a production process manager for Node.js applications. It manages and monitors applications, ensuring they run reliably.

### Install PM2

```bash
npm i -g pm2
```

### Start Your Application

```bash
cd /var/www/<your_project>
pm2 start npm --name "your_app_name" -- start
pm2 save
```

**Note:** There is a space before `-- start`

### Manage PM2 Processes

```bash
pm2 ls                      # List all processes
pm2 logs your_app_name      # View logs
pm2 restart your_app_name   # Restart application
pm2 stop your_app_name      # Stop application
pm2 delete your_app_name    # Remove from PM2
```

### Test Backend

```bash
curl http://localhost:8016
```

---

## MongoDB Installation

### Step 1: Import MongoDB Public Key

Install prerequisites:

```bash
sudo apt-get install gnupg curl
```

Import GPG key:

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg \
   --dearmor
```

### Step 2: Check Ubuntu Version

```bash
lsb_release -a
```

### Step 3: Create MongoDB List File

**For Ubuntu 24.04 (Noble):**

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
```

**For Ubuntu 22.04 (Jammy):**

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
```

**For Ubuntu 20.04 (Focal):**

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
```

### Step 4: Install MongoDB

```bash
sudo apt-get update
sudo apt-get install -y mongodb-org
```

### Step 5: Start MongoDB

```bash
sudo systemctl start mongod
```

If you receive an error:

```bash
sudo systemctl daemon-reload
sudo systemctl start mongod
```

### Step 6: Verify MongoDB Status

```bash
sudo systemctl status mongod
```

### Step 7: Enable MongoDB on Boot

```bash
sudo systemctl enable mongod
```

### Step 8: Connect to MongoDB

```bash
mongosh
```

---

## MongoDB Replica Set Configuration

### Step 1: Edit MongoDB Configuration

```bash
sudo nano /etc/mongod.conf
```

Add the following:

```yaml
replication:
   replSetName: "myReplicaSet"
```

### Step 2: Restart MongoDB

```bash
sudo systemctl restart mongod
```

### Step 3: Initiate Replica Set

```bash
mongosh
```

Inside the MongoDB shell:

```javascript
rs.initiate()
```

### Step 4: Allow MongoDB Port

```bash
sudo ufw allow 27017
sudo ufw allow 27017/tcp
sudo ufw reload
```

### Step 5: Test Connection

```bash
mongosh --port 27017
```

---

## SSL Certificate Setup

### Install Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
```

### Generate SSL Certificate

```bash
sudo certbot --nginx -d yourdomain.com
```

### Reload Nginx

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## Common Commands Reference

### MongoDB

```bash
sudo systemctl status mongod    # Check status
sudo systemctl start mongod     # Start MongoDB
sudo systemctl stop mongod      # Stop MongoDB
sudo systemctl restart mongod   # Restart MongoDB
```

### Nginx

```bash
sudo nginx -t                   # Test configuration
sudo systemctl restart nginx    # Restart Nginx
sudo nginx -s reload           # Reload configuration
```

### PM2

```bash
pm2 ls                         # List processes
pm2 logs <name>                # View logs
pm2 restart <name>             # Restart app
pm2 stop <name>                # Stop app
pm2 delete <name>              # Remove app
pm2 save                       # Save process list
```

---

## Additional Resources

- [MongoDB Official Documentation](https://www.mongodb.com/docs/manual/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [PM2 Documentation](https://pm2.keymetrics.io/docs/)
- [NVM GitHub Repository](https://github.com/nvm-sh/nvm)

---

## License

This documentation is provided as-is for educational and reference purposes.

## Contributing

Feel free to submit issues or pull requests to improve this documentation.
