# AWS Setup

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
   -  `projectname-storage`
   -  `projectname-website`
-  Ensure all buckets are created in the **US-West (Oregon)** region.

---

### 7. Configure Bucket Permissions

-  While creating each bucket, **uncheck** the option for **"Block all public access"**.
- This allows public access to the buckets, which is necessary for hosting static websites.

---

### 8. Enable Static Website Hosting

-  For the `admin` and `website` buckets:
   -  Go to the bucket’s **Properties** tab.
   -  Scroll down to **Static Website Hosting** and enable it.
   -  Set `index.html` as both the **Index Document** and **Error Document**.
   -  Update the bucket policy to:

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": "*",
			"Action": "s3:GetObject",
			"Resource": "<bucket-arn>/*"
		}
	]
}
```

---

### 9. Update Storage Bucket Policy

-  For the `storage` bucket, update the bucket policy to allow public read access:

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "PublicReadGetObject",
			"Effect": "Allow",
			"Principal": "*",
			"Action": "s3:GetObject",
			"Resource": "<bucket-arn>/*"
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
   -  `projectname-storage`
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
      -  `domain.com`
      -  `www.domain.com`

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

# Server Setup Instructions

## 1. Update and Upgrade Server

```bash
sudo apt update -y
```

```bash
sudo apt upgrade -y
```

## 3. Install Node.js and npm

### Install Node.js:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash
```

```bash
sudo apt-get install -y nodejs
```

### Install npm:

```bash
sudo npm install -g npm
```

## 4. Install MongoDB

Reference: [Install MongoDB Community Edition on Ubuntu - MongoDB Manual v8.0 - MongoDB Docs](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/)

### Create MongoDB User:

#### Connect to the database using the mongo shell
```bash
mongosh
```

#### in mongo shell
```bash
use admin
```

#### Create a user with a unique password
```bash
db.createUser({  user: "single-solution",  pwd: "password-unescaped-characters",  roles: [{ role: "root", db: "admin" }]})
```

#### Exit from mongosh
```bash
exit
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
sudo systemctl restart mongod
```

### Test Authentication:

#### Try connecting to MongoDB with authentication enabled by using:

```bash
mongosh -u "single-solution" -p "password" --authenticationDatabase "admin" --host localhost --port 27017
```

#### Exit from mongosh
```bash
exit
```

### Allow MongoDB port in firewall:

```bash
sudo ufw allow 27017
```

```bash
sudo systemctl restart mongod
```

### Enable port in aws security group

#### Follow these steps to enable port 27017 in the AWS Security Group:

<blockquote>
 
#### 1. Login

-  Log in to your AWS Management Console.

#### 2. Go to the AWS EC2 Instance

-  Navigate to the **EC2 Dashboard** from the AWS Services menu.

#### 3. Select EC2 Instance

-  From the list of instances, select the EC2 instance you want to configure.

#### 4. Go to the Security Tab

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

### Test MongoDB URL from firewall

#### Use the AWS Public IPv4 address
```bash
sudo nc -zv <server-ip> 27017
```

### MongoDB Compass URL

```bash
mongodb://<username>:<password>@<ip>:27017/?authSource=admin&retryWrites=true&w=majority
```

### MongoDB server env file URL

```bash
mongodb://single-solution:<password>@localhost:27017/database?authSource=admin&retryWrites=true&w=majority
```

## 5. Configure Swap Space

Reference: [Add Swap Space on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-20-04)

## 6. Create And Configure Nginx

### Install Nginx

Follow this link: [How To Install Nginx on Ubuntu 20.04 | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04)

After Nginx is installed, enable it:

```bash
sudo ufw enable
```

```bash
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
```

```bash
sudo cp default server
```

```bash
sudo nano server
```

Replace the server file content with this
                                  
```nginx
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
    location ~* \.(jpg|jpeg|png|gif|webp)$ {
        add_header Cache-Control "max-age=604800, public";
    }
}

```

Create a separate file for the static website if it has not been added to the S3 bucket

```nginx
server {
    server_name api.domain.com www.api.domain.com;

    root /root/workspace/website;
    index index.html index.htm;

    location / {
       try_files $uri $uri/ /index.html = 404;
    }

    # Cache-Control for images
    location ~* \.(jpg|jpeg|png|gif|webp)$ {
        add_header Cache-Control "max-age=604800, public";
    }
}

```
#### Restart Server
```bash
sudo systemctl restart nginx
```

### Allow nginx permissions

#### Create a directory workspace and allow permissions
#### After creating the directory, set permissions
```bash
sudo chown -R www-data:www-data /home/ubuntu/workspace
```
```bash
sudo chmod -R 755 /home/ubuntu/workspace
```
```bash
sudo chmod +x /home
```
```bash
sudo chmod +x /home/ubuntu
```
```bash
sudo chmod +x /home/ubuntu/workspace
```
```bash
sudo chmod +x /home /home/ubuntu /home/ubuntu/workspace
```

#### Restart Server
```bash
sudo systemctl restart nginx
```

```bash
sudo chown -R ubuntu:www-data /home/ubuntu/workspace
```

## 7. Install SSL Certificates

Reference: [Certbot Instructions](https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal)

Command:

```bash
sudo snap install --classic certbot
```
```bash
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
```bash
sudo certbot --nginx
```
```bash
sudo ufw allow 443
```

## 8. PM2 Configuration

PM2 helps to automatically restart the server in case of a file change or server crash

### Install PM2 service

```bash
sudo npm install pm2 -g
```

### Setup PM2 service

Inside the server directory `/home/ubuntu/workspace/server` create a new file `ecosystem.config.js`:

```bash
cd /home/ubuntu/workspace/server
```
```bash
sudo nano ecosystem.config.js
```

Replace this content in the file

```javascript
module.exports = {
    apps: [{
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
    }, ],
};
```

### Start PM2 service

```bash
pm2 start ecosystem.config.js
```

## 9. Install Redis (if required)

Commands:
```bash
sudo apt install redis
```

```bash
sudo systemctl enable redis-server
```

```bash
sudo systemctl status redis
```

## 10. Set up Cron Jobs (if required)

Reference: [Cron Setup](https://snapshooter.com/learn/linux/cron)

**Note:** Ensure that for JS, crontab has a separate bash file for proper syntax.

## 11. Deploy Everything to the panels just created

### Add this to the server env

```bash
#  //////////////////////////////
#  // AWS Credentials
#  //////////////////////////////
# AWS Credentials
AWS_S3_ACCESS_KEY_ID                        = AKIAVPDFRENTDDBJIX2F
AWS_S3_SECRET_ACCESS_KEY                    = F93TW2B/QQ9LgAzzr4Doot9ZDHIG4qkE
AWS_S3_AWS_REGION                           = us-west-2
AWS_S3_PROD_ADMIN_BUCKET                    = domain-admin
AWS_S3_PROD_STORAGE_BUCKET                  = domain-storage
AWS_S3_PROD_WEBSITE_BUCKET                  = domain-website
AWS_S3_DEV_ADMIN_BUCKET                     = domain-admin-dev
AWS_S3_DEV_STORAGE_BUCKET                   = domain-storage-dev
AWS_S3_DEV_WEBSITE_BUCKET                   = pdomain-website-dev
AWS_CLOUDFRONT_PROD_ADMIN_DISTRIBUTION_ID   = EMHSEASHXKP
AWS_CLOUDFRONT_PROD_WEBSITE_DISTRIBUTION_ID = E7UWEDFGZ8O
AWS_CLOUDFRONT_DEV_ADMIN_DISTRIBUTION_ID    = EASDAQON4UE
AWS_CLOUDFRONT_DEV_WEBSITE_DISTRIBUTION_ID  = ED75I2ERWAP

# EC2 Connection Details for Production
EC2_PROD_USER       = ubuntu
EC2_PROD_HOST       = ec2-1-1-1-1.us-west-2.compute.amazonaws.com
EC2_PROD_KEY_PATH   = keyfile.pem absolute path
EC2_PROD_REMOTE_DIR = /home/ubuntu/workspace/server

# EC2 Connection Details for Development (if different)
EC2_DEV_USER       = ubuntu
EC2_DEV_HOST       = ec2-1-1-1-1.us-west-2.compute.amazonaws.com
EC2_DEV_KEY_PATH   = keyfile.pem absolute path
EC2_DEV_REMOTE_DIR = /home/ubuntu/workspace/server
```

#### Go to the Server Folder for the project, paste the provided [deploy.js](./deploy.js) script
Add more flags after the environment `--server --website --admin`. If flags are provided, it will update that one specifically; otherwise, if no flag is provided, it will update all.

- Example Deployment `--dev for beta, --prod for live domain`
```bash
node deploy.js --dev --website
```
