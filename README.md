# This is an end-to-end, enterprise-grade deployment guide. 

## It takes you from a blank server to a running Java web application, applying best practices for security and reliability.

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

This blueprint gives you a clean, secure, and easily maintainable server. Would you like me to show you the next step in enterprise architecture, which is setting up an Nginx reverse proxy so users can access your site on standard port `80` without typing `:8080` in the URL?
