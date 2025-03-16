# Server Setup Instructions

## 1. Update and Upgrade Server

```bash
sudo apt update -y
sudo apt upgrade -y
```

## 3. Install Node.js and npm

### Install Node.js:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash

sudo apt-get install -y nodejs
```

### Install npm:

```bash
sudo npm install -g npm
```

## 4. Install MongoDB

Reference: [Install MongoDB Community Edition on Ubuntu - MongoDB Manual v8.0 - MongoDB Docs](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/)

### Create MongoDB User:

```bash
# Connect to database using mongo shell
mongosh

# in mongo shell
use admin

db.createUser({  user: "username",  pwd: "password-unescaped-characters",  roles: [{ role: "root", db: "admin" }]})
```

### Configure MongoDB authentication:

```bash
sudo nano /etc/mongod.conf
```

Add the following configuration:

```yaml
net:
   port: 27017
   bindIp: 0.0.0.0 # allows connections from all IPs

security:
   authorization: "enabled"
```

### Restart the server

```bash
sudo systemctl restart mongodb
```

### Test Authentication:

#### Try connecting to MongoDB with authentication enabled by using:

```bash
mongo -u "username" -p "password" --authenticationDatabase "admin" --host localhost --port 27017
```

### Allow mongodb port in firewall:

```bash
sudo ufw allow 27017
sudo systemctl restart mongodb
```

### Enable port in aws security group

#### Follow these steps to enable port 27017 in the AWS Security Group:

<blockquote>
#### 1. Login
- Log in to your AWS Management Console.

#### 2. Go to AWS EC2 Instance

-  Navigate to the **EC2 Dashboard** from the AWS Services menu.

#### 3. Select EC2 Instance

-  From the list of instances, select the EC2 instance you want to configure.

#### 4. Go to Security Tab

-  In the instance details page, click on the **Security** tab.
-  Locate and select the **security group** attached to the instance.

#### 5. Add New Rule

-  Click on **Edit inbound rules**.
-  Add a new rule with the following details:
   -  **Type**: `Custom TCP`
   -  **Port Range**: `27017`
   -  **Source**: `0.0.0.0/0` (or restrict to a specific IP range for better security)

#### 6. Save Changes

-  Click **Save rules** to apply the changes.
</blockquote>

### Test Mongodb url from firewall

```bash
# use aws Public IPv4 address
sudo nc -zv <server-ip> 27017
```

### MongoDB Compass URL

```bash
mongodb://<username>:<password>@<ip>:27017/?authSource=admin&retryWrites=true&w=majority
```

## 5. Configure Swap Space

Reference: [Add Swap Space on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-20-04)

## 6. Create And Configure Nginx

### Install Nginx

Follow this link: [How To Install Nginx on Ubuntu 20.04 | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04)

After Nginx is installed, enable it:

```bash
sudo ufw enable
sudo ufw allow openSSH
```

### Configure Nginx to upload files of larger size

```bash
sudo nano /etc/nginx/nginx.conf
```

Add the following configuration under the `http` block:

```nginx
http {
    # current code block

    # max file size to be uploaded to server
    client_max_body_size 1G;
}
```

### Create Nginx files

Nginx by default has a server file with permission to server the website. We are using that file and making a copy of it

```bash
cd /etc/nginx/sites-enabled/

sudo cp default server

sudo nano server

```

Replace the server file content with this

<blockquote>                                   
server {
    server_name api.domain.com www.api.domain.com;

    root /root/workspace/server;
    index index.html index.htm;

    location / {
       proxy_pass http://127.0.0.1:5010;
       proxy_http_version 1.1;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
       proxy_set_header Host $host;
    }

    # Cache-Control for images
    # Cache-Control for images
    location ~* \.(jpg|jpeg|png|gif|webp)$ {
        add_header Cache-Control "max-age=604800, public";
    }

</blockquote>

### Allow nginx permissions

```bash
sudo systemctl restart nginx

# create a directory workspace and allow permissions
# after creating the directory set permissions
sudo chown -R www-data:www-data /home/ubuntu/workspace
sudo chmod -R 755 /home/ubuntu/workspace
sudo chmod +x /home
sudo chmod +x /home/ubuntu
sudo chmod +x /home/ubuntu/workspace
sudo chmod +x /home /home/ubuntu /home/ubuntu/workspace

sudo systemctl restart nginx

sudo chown -R ubuntu:www-data /home/ubuntu/workspace
```

## 7. Install SSL Certificates

Reference: [Certbot Instructions](https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal)

Command:

```bash
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx
sudo ufw allow 443
```

## 8. PM2 Configuration

PM2 helps to automatically restart the server in case of file change or server crash

### Install PM2 service

```bash
sudo npm install pm2 -g
```

### Setup PM2 service

Inside server directory `/home/ubuntu/workspace/serer` create a new file `ecosystem.config.js`:

```bash
cd /home/ubuntu/workspace/serer

sudo nano ecosystem.config.js
```

Replace this content in new file

```javascript
module.exports = {
	apps: [
		{
			name: "server-name",
			script: "./index.js", // replace with your start file, like 'app.js'
			watch: true,
			ignore_watch: ["node_modules", "assets"], // these will ignored to restart the server
			watch_options: {
				followSymlinks: false,
			},
			env: {
				NODE_ENV: "development",
				// Other environment variables
			},
			env_production: {
				NODE_ENV: "production",
				// Other production environment variables
			},
		},
	],
};
```

### Start PM2 service

```bash
pm2 start ecosystem.config.js
```

## 9. Install Redis

Commands:

```bash
sudo apt install redis
sudo systemctl enable redis-server
sudo systemctl status redis
```

## 10. Setup Cron Jobs

Reference: [Cron Setup](https://snapshooter.com/learn/linux/cron)

**Note:** Ensure that for JS, crontab has a separate bash file for proper syntax.

---

## AWS Infrastructure Setup Guide

This guide will walk you through setting up your AWS infrastructure step by step. Follow each step carefully to ensure everything is configured correctly.

---

### 1. Set AWS Region to US-West (Oregon)

-  Open the **AWS Management Console**.
-  In the top-right corner, click on the **Region dropdown**.
-  Select **US-West (Oregon)** as your region.

---

### 2. Create a Secret Access Key

-  Navigate to **IAM (Identity and Access Management)**.
-  Go to **Users** and select your user account.
-  Under the **Security Credentials** tab, click **Create Access Key**.
-  Save the **Access Key ID** and **Secret Access Key** securely. You’ll need these for programmatic access to AWS services.

---

### 3. Launch an EC2 Instance

-  Go to the **EC2 Dashboard**.
-  Click **Launch Instance**.
-  Configure the instance as follows:
   -  **Name**: `projectname-server`
   -  **AMI (Amazon Machine Image)**: Select **Ubuntu** (latest version).
   -  **Instance Type**: Choose **t3** (general-purpose, cost-effective tier).
   -  **Key Pair**: Create a new key pair named `projectname` and download the `.pem` file.
   -  **Network Settings**: Ensure **HTTP traffic** is allowed in the security group.
-  Click **Launch Instance**.

---

### 4. Create and Associate an Elastic IP

-  Go to the **Elastic IPs** section in the EC2 Dashboard.
-  Click **Allocate Elastic IP Address**.
-  Once created, select the Elastic IP and click **Associate Elastic IP Address**.
-  Choose the EC2 instance you launched earlier (`projectname-server`) and associate the IP.

---

### 5. Set Up Your Server

-  Follow the **Web Hosting Guide** to configure your server. This typically involves:
   -  Installing a web server (e.g., Nginx or Apache).
   -  Setting up your application.
   -  Configuring firewall rules.

---

### 6. Create S3 Buckets

-  Go to the **S3 Dashboard**.
-  Click **Create Bucket** and create the following buckets:
   -  `projectname-admin`
   -  `projectname-server`
   -  `projectname-website`
-  Ensure all buckets are created in the **US-West (Oregon)** region.

---

### 7. Configure Bucket Permissions

-  While creating each bucket, **uncheck** the option for **"Block all public access"**.
-  This allows public access to the buckets, which is necessary for static website hosting.

---

### 8. Enable Static Website Hosting

-  For the `admin` and `website` buckets:
   -  Go to the bucket’s **Properties** tab.
   -  Scroll down to **Static Website Hosting** and enable it.
   -  Set `index.html` as both the **Index Document** and **Error Document**.
   -  Update the bucket policy for the `admin` bucket:

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": "*",
			"Action": "s3:GetObject",
			"Resource": "arn:aws:s3:::undrright-admin/*"
		}
	]
}
```

---

### 9. Update Server Bucket Policy

-  For the `server` bucket, update the bucket policy to allow public read access:

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "PublicReadGetObject",
			"Effect": "Allow",
			"Principal": "*",
			"Action": "s3:GetObject",
			"Resource": "arn:aws:s3:::undrright-server/*"
		}
	]
}
```

---

### 10. Create CloudFront Distributions

-  Go to the **CloudFront Dashboard**.
-  Click **Create Distribution**.
-  Create distributions for all three buckets:
   -  `projectname-admin`
   -  `projectname-server`
   -  `projectname-website`
-  Configure each distribution with the following settings:
   -  **Origin Domain**: Select the corresponding S3 bucket.
   -  **Viewer Protocol Policy**: Redirect HTTP to HTTPS.

---

### 11. Request an ACM Certificate

-  Go to the **ACM (AWS Certificate Manager)** Dashboard.
-  Request a certificate in the **US-East (Virginia)** region (required for CloudFront).
-  Add all domain names (e.g., `domain.com`, `www.domain.com`).
-  Choose **DNS Validation** and click **Request**.

---

### 12. Add CNAME Records to DNS

-  After requesting the certificate, you’ll receive **CNAME records**.
-  Go to your DNS provider (e.g., Route 53, GoDaddy) and add these CNAME records.
-  Wait for the certificate status to change to **Issued**.

---

### 13. Update CloudFront with SSL Certificates

-  Once the certificate is issued, go back to your CloudFront distributions.
-  Update each distribution to use the new SSL certificate.

---

### 14. Add Alternate Domain Names

-  Edit the `website` and `admin` CloudFront distributions:
   -  Add **Alternate Domain Names** like:
      -  `beta.undrright.com`
      -  `www.beta.undrright.com`

---

### 15. Add DNS A Records

-  Go to your DNS provider and add the following **A Records**:
   -  `api.domain.com` and `www.api.domain.com` → **EC2 Instance IP Address**
   -  `domain.com` and `www.domain.com` → **CloudFront Distribution Domain Name**
   -  `admin.domain.com` and `www.admin.domain.com` → **CloudFront Distribution Domain Name**

---

### 16. Invalidate CloudFront Cache

-  After completing the setup, invalidate the CloudFront cache to ensure the latest content is served:
   -  Go to the **CloudFront Distribution** → **Invalidations Tab**.
   -  Create a new invalidation with the path `/*`.
