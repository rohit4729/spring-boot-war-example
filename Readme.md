# 🚀 Deploy .war file to Tomcat Server From Jenkins
---
## 🔌 Prerequisites & Plugin Installation

Before configuring the job, ensure the following plugins are installed via **Manage Jenkins > Plugins**:

* **Maven Integration**: Required for creating Maven-specific project jobs.
* **Deploy to Container**: Enables Jenkins to move the `.war` file to the Tomcat server.
* **Build Pipeline**: Provides a visual representation of the automation flow.

---

## ⚙️ Configuration Steps

### Step 1: Global Tool Configuration
To ensure Jenkins can compile the Java code, we must link Maven:

1.  Navigate to **Manage Jenkins** > **Tools**.
2.  Locate the **Maven** section.
3.  Click **Add Maven**, give it a name (e.g., `MAVEN_HOME`), and select **Install automatically** or provide the local path.

### Step 2: Create and Configure the Jobs

### 2.1 We will create a specialized job to pull and test the source code:

1.  **New Item**: Create a new **Maven Project** named `helloWorld-test`.
2.  **Source Code Management**: 
    * Select **Git**.
    * **Repository URL**: [Insert your GitHub URL here]
    * **Branches to build**: `*/main` (or your preferred branch).
3.  **Build Section**:
    * **Root POM**: `pom.xml`
    * **Goals and options**: `test` 
    *(This ensures the code is validated before we move to the packaging stage.)*

### 2.2 The Build Job (`helloWorld-build`)
This job is responsible for compiling the code and packaging it into a `.war` file.


To complete the lifecycle of the application, we need to create separate jobs for building and post-build tasks.


1.  **New Item**: Create a **Maven Project** named `helloWorld-build`.
2.  **Source Code Management**:
    * Select **Git**.
    * **Repository URL**: [Insert your GitHub URL here]
3.  **Build Section**:
    * **Root POM**: `pom.xml`
    * **Goals and options**: `install`
    *(The `install` goal packages the .war file and moves it to the local Maven repository.)*

### 2.3: Placeholder Jobs
Create the following two **Freestyle Project** items. We will leave the configuration empty for now as they will serve as downstream stages in our pipeline:

1.  **Job 1**: `helloWorld-Deploy-test` (Placeholder for Tomcat deployment).
2.  **Job 2**: `helloWorld-Deploy-Prod` (Placeholder for post-deployment alerts).

### Step 3: Connecting the Pipeline (Chaining Jobs)

Now we link the jobs together so that a success in one stage automatically triggers the next.

#### 3.1: Automated Chain (Test ➔ Build ➔ Deploy-test)
We use the **Post-build Actions** to create a downstream flow:

1.  **In `helloWorld-test`**:
    * Go to **Post-build Actions** > **Build other projects**.
    * Project to build: `helloWorld-build`.
    * Condition: *Trigger only if build is stable*.
2.  **In `helloWorld-build`**:
    * Go to **Post-build Actions** > **Build other projects**.
    * Project to build: `helloWorld-Deploy-test`.
    * Condition: *Trigger only if build is stable*.

#### 3.2: Manual Production Gate (Deploy-test ➔ Deploy-Prod)
To prevent accidental deployments to Production, we use a manual trigger:

1.  **In `helloWorld-Deploy-test`**:
    * Go to **Post-build Actions**.
    * Select **Build other projects (manual step)**.
    * Project to build: `helloWorld-Deploy-Prod`.



## 📈 Current Pipeline Logic


| Job Name | Trigger Type | Action |
| :--- | :--- | :--- |
| `helloWorld-test` | Source Code Change | Runs Maven Tests |
| `helloWorld-build` | Automatic (Success) | Runs Maven Install (Generates .war) |
| `helloWorld-Deploy-test` | Automatic (Success) | Deploys to Staging Tomcat |
| `helloWorld-Deploy-Prod` | **Manual Approval** | Deploys to Production Tomcat |
---
### Step 4: Visualizing the Pipeline (Build Pipeline View)

To monitor the status of our CI/CD flow in real-time, we will create a dedicated Pipeline View.

1.  Click the **"+" (New View)** tab on the Jenkins dashboard.
2.  **View Name**: Enter `HelloWorld-Pipeline`.
3.  **Type**: Select **Build Pipeline View**.
4.  **Configuration**:
    * Under **Layout**, find the **Select Initial Job** dropdown.
    * Select `helloWorld-test` (this is the "Upstream" job that kicks off the process).
5.  Click **OK**.



## 📊 Pipeline Overview
The resulting view will display the sequence of jobs, their current status (Green for success, Red for failure), and the manual trigger button for the Production deployment.

> **Note**: This view makes it easy to track which version of the code is currently in "Test" vs "Production" and allows team leads to manually approve the final deployment step with a single click.
> <img width="1920" height="918" alt="Screenshot (147)" src="https://github.com/user-attachments/assets/b9d1e467-7830-40f0-bd39-0f9e5f98d9b7" />

---

## 🌐 Step 6: Target Server Setup (Ubuntu 24.04 LTS)

Repeat these steps for both the **Test** and **Production** instances to ensure they are ready to host the `.war` files.

### 6.1: Environment & Binary Installation
First, we install Java 17 and set up the directory structure for Tomcat 9.

```bash
# Update and Install Java
sudo apt-get update
sudo apt install -y openjdk-17-jdk

# Create dedicated Tomcat user
sudo useradd -m -U -d /opt/tomcat9 -s /bin/false tomcat

# Download and Extract Tomcat 9
cd /tmp
wget [https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.113/bin/apache-tomcat-9.0.113.tar.gz](https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.113/bin/apache-tomcat-9.0.113.tar.gz)
sudo mkdir -p /opt/tomcat9
sudo tar -xf apache-tomcat-9.0.113.tar.gz -C /opt/tomcat9 --strip-components=1

# Set Permissions
sudo chown -R tomcat:tomcat /opt/tomcat9
sudo chmod +x /opt/tomcat9/bin/*.sh

```
### 6.2: Configure Tomcat as a System Service
Create a systemd service file to manage Tomcat with systemctl.

Create the file: sudo nano /etc/systemd/system/tomcat.service   

Paste the configuration:
```bash
[Unit]
Description=Apache Tomcat 9 Web Application Container
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"
Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/./urandom -Djava.awt.headless=true"
Environment="CATALINA_BASE=/opt/tomcat9"
Environment="CATALINA_HOME=/opt/tomcat9"
Environment="CATALINA_PID=/opt/tomcat9/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=/opt/tomcat9/bin/startup.sh
ExecStop=/opt/tomcat9/bin/shutdown.sh

[Install]
WantedBy=multi-user.target
```

### 6.3: Start the server

```bash
sudo systemctl daemon-reload
sudo systemctl start tomcat
sudo systemctl enable tomcat

```
### 6.4: GUI Access & Remote Management
By default, Tomcat restricts access to the Manager App. We must unlock it for Jenkins to deploy files.

1. Add Users: Edit /opt/tomcat9/conf/tomcat-users.xml and add:
 ```bash
<user username="xxxxx" password="****" roles="manager-gui,manager-script,manager-jmx,manager-status"/>
```
2. Unlock Remote Access: Comment out the <Valve.../> (Remote Address Filter) in these two files:

/opt/tomcat9/webapps/manager/META-INF/context.xml

/opt/tomcat9/webapps/host-manager/META-INF/context.xml

3. Restart: sudo systemctl restart tomcat
### 6.5: Verification
Open your browser and navigate to: http://<INSTANCE_IP>:8080

Click Manager App and login with:

Username: xxxxx

Password: *****

<img width="1920" height="1080" alt="Screenshot (143)" src="https://github.com/user-attachments/assets/65f1efbf-67b3-44cd-8582-f8c582293677" />
