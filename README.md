# Tomcat Installation and Improvements V-0.0.1

This is an end-to-end, enterprise-grade deployment guide.It takes you from a blank server to a running Java web application, applying best practices for security and reliability.

Here is your "Pro" playbook for deploying Petclinic to Tomcat on a fresh Linux server.

---

### Phase 1: Connect & Prepare the Server

**1. SSH into the Server**
Open your terminal and connect using your private key. 
* *Ubuntu default user:* `ubuntu`
* *RHEL/Amazon Linux default user:* `ec2-user`
```bash
ssh -i /path/to/your-key.pem ubuntu@YOUR_SERVER_IP
```

**2. Update the System and Install Prerequisites**
A pro always starts by updating the package manager and installing Git, Java, and Maven.

* **For Ubuntu/Debian:**
  ```bash
  sudo apt update -y
  sudo apt install -y default-jdk maven git wget tar
  ```
* **For RHEL/CentOS/Amazon Linux:**
  ```bash
  sudo dnf update -y
  sudo dnf install -y java-11-openjdk-devel maven git wget tar
  ```

---

### Phase 2: Secure Infrastructure Setup

**1. Create the Service Account**
Never run web servers as root. We create a dedicated `tomcat` user with no login privileges (`/bin/false`) to contain any potential security breaches.
```bash
sudo useradd -m -U -d /opt/tomcat -s /bin/false tomcat
```

**2. Download and Extract Tomcat**
We download it to a temporary folder, then extract it into `/opt`, which is the standard Linux directory for third-party software.
```bash
cd /tmp
wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.65/bin/apache-tomcat-9.0.65.tar.gz
sudo tar -xf apache-tomcat-9.0.65.tar.gz -C /opt/
```

**3. Create the Generic Symlink**
This makes future upgrades seamless. If you ever upgrade Tomcat, you just change where this link points, and you don't have to rewrite any of your system scripts.
```bash
sudo ln -s /opt/apache-tomcat-9.0.65 /opt/tomcat
```

**4. Lock Down Permissions**
Give the `tomcat` user ownership of the files and make the startup scripts executable.
```bash
sudo chown -R tomcat:tomcat /opt/apache-tomcat-9.0.65
sudo chown -R tomcat:tomcat /opt/tomcat
sudo chmod -R u+x /opt/tomcat/bin
```

---

### Phase 3: Systemd Daemon Configuration

We want Linux to manage Tomcat so it starts automatically if the server reboots and restarts if it crashes.

**1. Create the Service File**
```bash
sudo nano /etc/systemd/system/tomcat.service
```

**2. Paste the Configuration**
*(Note: If you are on RHEL, change `default-java` to the actual Java path, usually `/usr/lib/jvm/jre`)*
```ini
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment="JAVA_HOME=/usr/lib/jvm/default-java"
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

[Install]
WantedBy=multi-user.target
```
*Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).*

**3. Enable and Start the Service**
Tell the system to read the new file, enable it on boot, and start it now.
```bash
sudo systemctl daemon-reload
sudo systemctl enable tomcat
sudo systemctl start tomcat
```

---

### Phase 4: Build and Deploy the Application

**1. Clone the Source Code**
Pull the Petclinic code from GitHub to your home directory.
```bash
cd ~
git clone https://github.com/jaiswaladi246/Petclinic.git
```

**2. Compile and Package the App**
Use Maven to download dependencies, compile the Java code, and create the `.war` deployment file.
```bash
cd Petclinic
mvn clean package
```
*(Wait for the "BUILD SUCCESS" message).*

**3. Deploy to Tomcat**
Force-copy the newly built `.war` file directly into Tomcat's `webapps` folder, then give the `tomcat` user ownership so the server can read and unpack it.
```bash
sudo cp target/petclinic.war /opt/tomcat/webapps/
sudo chown tomcat:tomcat /opt/tomcat/webapps/petclinic.war
```

---

### Phase 5: Verification

Tomcat monitors the `webapps` folder and will automatically extract `petclinic.war` into a running application folder.

Open your web browser and navigate to:
**`http://YOUR_SERVER_IP:8080/petclinic`**

> **Pro Tip for the Cloud:** If the site doesn't load, the number one culprit is the Cloud Firewall. Make sure your AWS Security Group (or equivalent) has an Inbound Rule allowing traffic on **Custom TCP Port 8080** from anywhere (`0.0.0.0/0`).

This blueprint gives you a clean, secure, and easily maintainable server. 

The next step in enterprise architecture, which is setting up an Nginx reverse proxy so users can access your site on standard port `80` without typing `:8080` in the URL.
Setting up a reverse proxy is what separates a development server from a production-ready environment. 

By default, Tomcat runs on port `8080`. However, web browsers expect standard web traffic to flow over port `80` (HTTP) or `443` (HTTPS). Instead of running Tomcat as the root user to bind it to port 80 (which is a massive security risk), we put a dedicated web server like **Nginx** in front of it. 



Nginx will listen on port 80, intercept the web traffic, and securely forward it to Tomcat on port 8080 behind the scenes. 

Here is your enterprise guide to setting up Nginx as a reverse proxy for your Petclinic application.

---

### Step 1: Install Nginx

Connect to your server and install the Nginx package.

**For Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install -y nginx
```

**For RHEL/CentOS/Amazon Linux:**
```bash
sudo dnf install -y nginx
```

Once installed, ensure Nginx is set to start automatically on boot and start the service:
```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

---

### Step 2: Create the Nginx Configuration File

We need to tell Nginx to listen for traffic on port 80 and route it to our Tomcat application. 

Create a new configuration file. On Ubuntu, it is best practice to create this in the `sites-available` directory. (If you are on RHEL, you can create this directly in `/etc/nginx/conf.d/petclinic.conf`).

```bash
sudo nano /etc/nginx/sites-available/petclinic
```

Paste the following configuration into the file. Replace `YOUR_SERVER_IP` with your actual public IP address or domain name:

```nginx
server {
    listen 80;
    server_name YOUR_SERVER_IP; # Or your domain name like www.myapp.com

    # Route traffic to the Petclinic application
    location /petclinic/ {
        proxy_pass http://127.0.0.1:8080/petclinic/;
        
        # Pass essential headers to Tomcat so it knows where the request came from
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Optional: Redirect root domain traffic directly to the app
    location / {
        return 301 /petclinic/;
    }
}
```
*Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).*

**What are those headers doing?**
When Nginx forwards the traffic, Tomcat sees the request as coming from `127.0.0.1` (localhost). Those `proxy_set_header` lines ensure Nginx passes the *original* user's IP address and details through to Tomcat, which is vital for your application's logging and security.

---

### Step 3: Enable the Configuration (Ubuntu/Debian Only)

If you are on Ubuntu, Nginx uses a symlink system to enable sites. You must link the file you just created into the `sites-enabled` folder:

```bash
sudo ln -s /etc/nginx/sites-available/petclinic /etc/nginx/sites-enabled/
```

*Pro-tip: It is also a good idea to remove the default Nginx welcome page so it doesn't conflict with your app:*
```bash
sudo rm /etc/nginx/sites-enabled/default
```

---

### Step 4: Test and Restart Nginx

A professional sysadmin *never* restarts a web server without testing the configuration first. If there is a typo in your file, restarting will crash the server.

Run the Nginx configuration test:
```bash
sudo nginx -t
```
*Look for this output:*
`nginx: configuration file /etc/nginx/nginx.conf syntax is ok`
`nginx: configuration file /etc/nginx/nginx.conf test is successful`

If the test passes, safely reload Nginx to apply the changes:
```bash
sudo systemctl reload nginx
```

---

### Step 5: Update Cloud Firewalls

Since you are now serving traffic over standard HTTP, you must ensure your Cloud Provider (AWS Security Groups, Azure NSG, etc.) allows inbound traffic on **Port 80 (HTTP)**. 

*(You can also safely close Port 8080 to the outside world now, as Nginx handles the public traffic and talks to Tomcat internally!)*

### Step 6: Verify the Proxy

Open your web browser and navigate to your server's IP address—**without the `:8080`**:

**`http://YOUR_SERVER_IP/`**

Nginx will instantly catch the request on port 80, realize you want the root path, redirect you to `/petclinic/`, and serve the Tomcat application seamlessly. 

Congratulations! You now have a robust, production-style web architecture. The natural next step for a modern application would be securing this connection with a free SSL certificate using **Certbot/Let's Encrypt** so your site runs on `HTTPS (Port 443)`.

Securing your application with HTTPS is a non-negotiable standard for any modern business solution. In enterprise system design, we use a pattern called **SSL Termination**. 

Instead of forcing your Tomcat server to handle the complex math of encrypting and decrypting traffic, we let Nginx handle it at the front door. Nginx decrypts the incoming HTTPS traffic and passes it as plain HTTP to Tomcat on the internal network. This keeps your Java app fast and focused solely on serving the application logic.



Here is how to set up a free, auto-renewing SSL certificate using Let's Encrypt and Certbot.

### ⚠️ The Golden Prerequisite: A Domain Name
Let's Encrypt **will not** issue a certificate for a bare IP address (like `172.31.x.x` or your public AWS IP). You must have a registered domain name (like `myapp.com` or `petclinic.yourname.com`) and an **A Record** in your DNS settings pointing to your server's public IP address.

Once your domain is pointing to your server, proceed with these steps:

---

### Step 1: Install Certbot
Certbot is the tool that automatically fetches and installs the Let's Encrypt certificate.

**For Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

**For RHEL/Amazon Linux:**
```bash
sudo dnf install -y certbot python3-certbot-nginx
```

### Step 2: Prepare the Firewall
Since HTTPS traffic flows over **Port 443**, you need to ensure your cloud provider's firewall (e.g., AWS Security Group) has an Inbound Rule allowing traffic on Port 443 from anywhere (`0.0.0.0/0`). 

### Step 3: Update Nginx Configuration (If Needed)
Certbot looks at your Nginx configuration to figure out which domain it is securing. Open the file we created in the last step:
```bash
sudo nano /etc/nginx/sites-available/petclinic
```
Ensure the `server_name` directive exactly matches your domain:
```nginx
server_name yourdomain.com www.yourdomain.com;
```
*(Save and exit, then run `sudo systemctl reload nginx` if you made changes).*

### Step 4: Run the Magic Command
This is where the automation shines. Run this command, and Certbot will communicate with Let's Encrypt, verify you own the domain, issue the certificate, and automatically rewrite your Nginx configuration to use it.

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

**During the setup, Certbot will ask you a few questions:**
1. Enter your email address (used for urgent renewal and security notices).
2. Agree to the Terms of Service.
3. **Important:** It will ask if you want to redirect all HTTP traffic to HTTPS. Choose **Redirect (Option 2)**. This ensures that even if someone types `http://`, they are forced into the secure `https://` connection.

### Step 5: Verify Auto-Renewal
Let's Encrypt certificates expire every 90 days, but Certbot sets up a background timer to renew them automatically before they expire. You can test that this automated process is working by running a "dry run":

```bash
sudo certbot renew --dry-run
```
If you see a success message, you never have to worry about renewing this certificate manually.

---

### The Final Result
If you open your `/etc/nginx/sites-available/petclinic` file now, you will see that Certbot added a bunch of lines pointing to your new cryptographic keys. 

Navigate to **`https://yourdomain.com/petclinic/`** in your browser, and you should see the secure padlock icon!

You now have a fully deployed, reverse-proxied, and SSL-secured infrastructure stack.
