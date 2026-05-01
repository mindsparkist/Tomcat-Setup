Launching servers in the cloud is a core skill for any system administrator or DevOps engineer. The terminology differs slightly between providers, but the underlying concepts (Compute, Networking, Security, Storage) are identical.

Here is your step-by-step guide to provisioning both Linux in AWS and Windows in Azure.

-----

### Part 1: Launching Ubuntu or RHEL in AWS (EC2)

In AWS, virtual servers are called **EC2 Instances** (Elastic Compute Cloud).

**Step 1: Access the EC2 Dashboard**

1.  Log in to the [AWS Management Console](https://console.aws.amazon.com/).
2.  In the top search bar, type **EC2** and select it.
3.  In the top right corner, verify your **Region** (e.g., us-east-1, ap-south-1). Choose the region closest to you or your users.
4.  Click the orange **Launch instance** button.

**Step 2: Name and OS Selection (AMI)**

1.  **Name:** Give your server a recognizable name (e.g., `Petclinic-Prod-Ubuntu`).
2.  **Application and OS Images (Amazon Machine Image):** \* Click on the **Ubuntu** or **Red Hat** tab.
      * Select the default architecture (usually 64-bit x86). If you are practicing, look for the badge that says **"Free tier eligible"** (like Ubuntu 22.04 LTS or RHEL 9).

**Step 3: Choose Instance Type**

1.  **Instance type:** This dictates your CPU and RAM. Select `t2.micro` or `t3.micro` (which are usually Free Tier eligible).

**Step 4: Create a Key Pair (Crucial)**
*You cannot connect to your Linux server without this.*

1.  Click **Create new key pair**.
2.  Name it (e.g., `my-aws-key`).
3.  Select **RSA** and **.pem** format (used for standard SSH terminals like Mac/Linux/Windows Terminal).
4.  Click **Create key pair**. *The file will immediately download to your computer. Keep it safe; AWS will not let you download it again.*

**Step 5: Configure Network Security**
This acts as your server's firewall (called a Security Group in AWS).

1.  Under **Network settings**, ensure **Create security group** is selected.
2.  Check the following boxes based on what you need:
      * **Allow SSH traffic from:** Set to **Anywhere** (0.0.0.0/0) or **My IP** (more secure). This allows you to use the terminal.
      * **Allow HTTP/HTTPS traffic from the internet:** Check these if you plan to run Nginx, Tomcat, or web apps on this server.

**Step 6: Launch**

1.  Leave storage at the default (usually 8GB or 10GB General Purpose SSD).
2.  Click the orange **Launch instance** button on the bottom right.
3.  Once successful, click the **Instance ID** link. Wait for the "Instance state" to say **Running**.

**How to Connect:**
Open your terminal on your personal computer, locate the `.pem` key you downloaded, and run:

```bash
# Secure the key file permissions first (Mac/Linux only)
chmod 400 my-aws-key.pem

# SSH into the server (Use 'ubuntu' for Ubuntu, or 'ec2-user' for RHEL)
ssh -i my-aws-key.pem ubuntu@YOUR_AWS_PUBLIC_IP
```

-----

### Part 2: Launching a Windows Server in Microsoft Azure

In Azure, virtual servers are simply called **Virtual Machines (VMs)**.

**Step 1: Access Virtual Machines**

1.  Log in to the [Azure Portal](https://www.google.com/search?q=https://portal.azure.com/).
2.  In the search bar at the top, type **Virtual Machines** and select it.
3.  Click **+ Create** -\> **Azure virtual machine**.

**Step 2: Project & Instance Details**

1.  **Resource Group:** This is a logical folder for your server. Click **Create new** and name it (e.g., `Windows-Servers-RG`).
2.  **Virtual machine name:** (e.g., `WinServer2022`).
3.  **Region:** Select a region close to you.
4.  **Image:** Open the dropdown and select **Windows Server 2022 Datacenter**.
5.  **Size:** Choose your CPU/RAM. Windows requires more power than Linux. Look for a `B2s` or `D2s_v3` size (at least 2 vCPUs and 4GB+ RAM) to ensure it doesn't freeze during setup.

**Step 3: Administrator Account (Crucial)**
Unlike Linux (which uses SSH keys by default), Windows primarily uses a username and password for remote desktop access.

1.  **Username:** Create a secure admin username (e.g., `sysadmin`). *Do not use "admin" or "administrator" as these are easily guessed by hackers.*
2.  **Password:** Create a complex password and write it down.

**Step 4: Inbound Port Rules (Security)**

1.  **Public inbound ports:** Select **Allow selected ports**.
2.  **Select inbound ports:** Check the box for **RDP (3389)**.
      * *Note: RDP (Remote Desktop Protocol) is what allows you to see the Windows desktop remotely. For production, you would restrict this to only your home/office IP address, but allowing it globally is fine for a quick test.*

**Step 5: Review and Create**

1.  Click the blue **Review + create** button at the very bottom.
2.  Azure will run a quick validation check. Once it passes, click **Create**.
3.  It will take a few minutes to deploy. Once done, click **Go to resource**.

**How to Connect:**

1.  On your VM's overview page in Azure, click the **Connect** button at the top, then select **RDP**.
2.  Download the RDP File provided.
3.  Open the downloaded file.
      * *If you are on a Mac, you will need to download the "Microsoft Remote Desktop" app from the Mac App Store first.*
4.  When prompted, enter the Username and Password you created in Step 3. Accept any certificate warnings, and you will see your Windows Server desktop load\!



# Tomcat Installation and Improvements V-0.0.2

#### Phase 1: Connect & Prepare the Server

**1. SSH into the Server**
Open your terminal and connect using your private key.
*   **Ubuntu default user:** `ubuntu`
*   **RHEL/Amazon Linux default user:** `ec2-user`
```bash
ssh -i /path/to/your-key.pem ubuntu@YOUR_SERVER_IP
```

**2. Update the System and Install Prerequisites**
A pro always starts by updating the package manager and installing Git, Java, and Maven.
*   **For Ubuntu/Debian:**
```bash
sudo apt update -y
sudo apt install -y default-jdk maven git wget tar
```
*   **For RHEL/CentOS/Amazon Linux:**
```bash
sudo dnf update -y
sudo dnf install -y java-11-openjdk-devel maven git wget tar
```

#### Phase 2: Secure Infrastructure Setup

**1. Create the Service Account (Fixed)**
Never run web servers as root. We create a dedicated `tomcat` user with no login privileges (`/bin/false`) to contain any potential security breaches. 
*(Note: We use `-M` instead of `-m` to ensure Linux does NOT create a physical home directory yet, keeping the path clear for our symlink later).*
```bash
sudo useradd -r -M -U -d /opt/tomcat -s /bin/false tomcat
```

**2. Download and Extract Tomcat**
We download it to a temporary folder, then extract it into `/opt`, which is the standard Linux directory for third-party software.
```bash
cd /tmp
wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.65/bin/apache-tomcat-9.0.65.tar.gz
sudo tar -xf apache-tomcat-9.0.65.tar.gz -C /opt/
```

**3. Create the Generic Symlink**
This makes future upgrades seamless. Because we didn't physically create an `/opt/tomcat` folder in Step 1, this link will be created perfectly. If you ever upgrade Tomcat, you just change where this link points.
```bash
sudo ln -s /opt/apache-tomcat-9.0.65 /opt/tomcat
```

**4. Lock Down Permissions (Fixed)**
Give the `tomcat` user ownership of the actual files, update the symlink's ownership, and ensure only the necessary startup scripts are executable.
```bash
sudo chown -R tomcat:tomcat /opt/apache-tomcat-9.0.65
sudo chown -h tomcat:tomcat /opt/tomcat
sudo chmod +x /opt/tomcat/bin/*.sh
```

***

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
