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
<img width="1920" height="1080" alt="Screenshot (146)" src="https://github.com/user-attachments/assets/1dbd4a05-bb44-4c55-9cb5-c4732854faea" />


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

## 🌐 Step 5: Target Server Setup (Ubuntu 24.04 LTS)

Repeat these steps for both the **Test** and **Production** instances to ensure they are ready to host the `.war` files.

### 5.1: Environment & Binary Installation
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
<img width="1920" height="1080" alt="Screenshot (149)" src="https://github.com/user-attachments/assets/01561713-d3b7-43e5-8ba9-1ef638a5b58d" />

### 5.2: Configure Tomcat as a System Service
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

### 5.3: Start the server

```bash
sudo systemctl daemon-reload
sudo systemctl start tomcat
sudo systemctl enable tomcat

```
### 5.4: GUI Access & Remote Management
By default, Tomcat restricts access to the Manager App. We must unlock it for Jenkins to deploy files.

1. Add Users: Edit /opt/tomcat9/conf/tomcat-users.xml and add:
 ```bash
<user username="xxxxx" password="****" roles="manager-gui,manager-script,manager-jmx,manager-status"/>
```
2. Unlock Remote Access: Comment out the <Valve.../> (Remote Address Filter) in these two files:

/opt/tomcat9/webapps/manager/META-INF/context.xml

/opt/tomcat9/webapps/host-manager/META-INF/context.xml

3. Restart: sudo systemctl restart tomcat
### 5.5: Verification
Open your browser and navigate to: http://<INSTANCE_IP>:8080

Click Manager App and login with:

Username: xxxxx

Password: *****

<img width="1920" height="1080" alt="Screenshot (143)" src="https://github.com/user-attachments/assets/65f1efbf-67b3-44cd-8582-f8c582293677" />

### Step 6: Artifact Management (Archiving & Permissions)

To ensure the deployment jobs can access the generated `.war` file, we must archive it and set the correct permissions.

#### 6.1: Plugin Installation
Ensure the **Copy Artifact** plugin is installed via **Manage Jenkins > Plugins**. This allows one job to pull files from another job's workspace.

#### 6.2: Configure `helloWorld-build` for Export
1. In General section select Permission to copy artifact and in the Projects to allow copy artifacts write helloWorld-Deploy-Prod.
2. We need to tell Jenkins to "save" the `.war` file and allow other jobs to "pick it up."

**6.2.1 Archive the Artifact**
1. Open the configuration for **`helloWorld-build`**.
2. Go to **Post-build Actions** > **Archive the artifacts**.
3. In **Files to archive**, enter: `**/*.war`
   *(The `**` ensures Jenkins finds the war file even if it is inside the /target folder.)*

**6.2.2 Set Access Permissions**
1. Scroll up to the **General** section of the same job.
2. Check the box for **Permission to Copy Artifact**.
3. In **Projects allowed to copy artifacts**, enter: `helloWorld-*`
   *(Using the wildcard `*` allows both your Test and Prod deployment jobs to access this build.)*
4. **Save** the configuration.



---

> **Note**: This setup prevents you from having to rebuild the code in every stage, which is a core principle of "Build Once, Deploy Many."

### Step 7: Finalizing the Test Deployment Job (`helloWorld-Deploy-test`)

This job acts as the delivery vehicle, pulling the stable build from the previous stage and pushing it to the Ubuntu Test server.

#### 7.1: Pulling the Build Artifact
1. Open the configuration for **`helloWorld-Deploy-test`**.
2. In the **Build Steps** section, select **Add build step** > **Copy artifacts from another project**.
3. **Project Name**: `helloWorld-build`.
4. **Which build**: *Latest successful build*.
5. **Artifacts to copy**: `**/*.war`.

#### 7.2: Deploying to Tomcat
We now configure the "Push" to the remote server.

**7.2.1 Re-Archiving (Optional but Recommended)**
* Add a **Post-build Action**: **Archive the artifacts**.
* Files to archive: `**/*.war`.
*(This keeps a record of exactly what was deployed to the test environment in this specific run.)*

**7.2.2 Container Deployment Configuration**
1. Add a **Post-build Action**: **Deploy war/ear to a container**.
2. **WAR/EAR files**: `**/*.war`.
3. **Context path**: `/app` 
   *(Your app will be accessible at `http://<IP>:8080/app`)*.
4. **Containers**: Click **Add Container** and select **Tomcat 9.x Remote**.
5. **Credentials**: Click **Add** and enter the Tomcat manager credentials (`rohit` / `raja28`) created in Step 6.
6. **Tomcat URL**: `http://<TEST_SERVER_PUBLIC_IP>:8080`.



---

## 🚀 How to Run the Pipeline
1. Go to the **HelloWorld-Pipeline** view created in Step 5.
2. Click the **Run** icon on the `helloWorld-test` job.
3. Watch as the green progress bar moves from **Test** ➔ **Build** ➔ **Deploy-test**.
4. Once `Deploy-test` finishes, visit `http://<TEST_IP>:8080/app` to see your application live!

### Step 8: Final Production Deployment (`helloWorld-Deploy-Prod`)

To ensure parity between environments, we copy the artifact that has already been successfully deployed to Test, rather than pulling a new build.

#### 8.1: Copy Verified Artifact
1. Open the configuration for **`helloWorld-Deploy-Prod`**.
2. In **Build Steps**, select **Copy artifacts from another project**.
3. **Project Name**: `helloWorld-Deploy-test`.
4. **Which build**: *Latest successful build*.
5. **Artifacts to copy**: `**/*.war`.

#### 8.2: Production Container Configuration
1. Add a **Post-build Action**: **Deploy war/ear to a container**.
2. **WAR/EAR files**: `**/*.war`.
3. **Context path**: `/app`.
4. **Containers**: Select **Tomcat 9.x Remote**.
5. **Credentials**: Use the same Tomcat manager credentials (`rohit` / `raja28`).
6. **Tomcat URL**: `http://<PRODUCTION_SERVER_PUBLIC_IP>:8080`.

---

## 🏁 Summary of the Complete Pipeline

Your automation is now fully configured! Here is the flow of data:

| Stage | Action | Artifact Source | Trigger |
| :--- | :--- | :--- | :--- |
| **Test** | Maven Test | GitHub Source | SCM Change |
| **Build** | Maven Install | GitHub Source | Auto (on Test success) |
| **Deploy-Test** | Push to Tomcat 1 | `helloWorld-build` | Auto (on Build success) |
| **Deploy-Prod** | Push to Tomcat 2 | `helloWorld-Deploy-test` | **Manual Approval** |



### 🔗 Verification
* **Test Environment**: `http://<TEST_IP>:8080/app`
* **Production Environment**: `http://<PROD_IP>:8080/app`

<img width="1920" height="1080" alt="Screenshot (144)" src="https://github.com/user-attachments/assets/04e42e4b-64f3-4657-9861-9d66f91dd3a2" />
<img width="1920" height="1080" alt="Screenshot (145)" src="https://github.com/user-attachments/assets/55709323-fa7f-46bf-876e-80741f8dc87b" />

---


### Step 9: Automating the Trigger (Poll SCM)

To make the pipeline truly "Continuous," we configure Jenkins to automatically check for new code changes in your repository.

1.  Open the configuration for the **`helloWorld-test`** job.
2.  Navigate to the **Build Triggers** section.
3.  Check the box for **Poll SCM**.
4.  In the **Schedule** text area, enter:
    ```cron
    H/2 * * * *
    ```
    * **What this does**: Jenkins will poll (check) your GitHub repository every **2 minutes** for any new commits. If it finds a change, it triggers the first job, which then kicks off the entire downstream pipeline.

<img width="1920" height="1080" alt="Screenshot (150)" src="https://github.com/user-attachments/assets/4f381580-9327-4f75-a11f-250d7f4f3de9" />

---

## 🏗 Full Architecture Overview

Now that the setup is complete, here is the automated flow of your DevOps ecosystem:



1.  **Developer** pushes code to GitHub.
2.  **Jenkins** (Step 9) detects the change via Polling.
3.  **Test Job** (Step 2) runs unit tests.
4.  **Build Job** (Step 2/2) packages the `.war` file and archives it.
5.  **Deploy-Test** (Step 7) pulls the artifact and deploys it to the Staging server.
6.  **Human Approval** is required to trigger the final **Deploy-Prod** (Step 8) to the live server.

<img width="3312" height="2017" alt="🏗 Full Architecture Overview - visual selection (1)" src="https://github.com/user-attachments/assets/fef17598-4312-4144-862c-932eb7e0a949" />

---

## 📝 Prerequisites Checklist
* [ ] Jenkins installed with **Maven**, **Deploy to Container**, and **Copy Artifact** plugins.
* [ ] Two Ubuntu 24.04 instances with **Tomcat 9** and **Java 17**.
* [ ] Tomcat `manager-script` credentials configured on both servers.
* [ ] GitHub Repository URL accessible by Jenkins.

<img width="3027" height="2233" alt="🏗 Full Architecture Overview - visual selection" src="https://github.com/user-attachments/assets/b4b8fe61-c202-4c9a-883c-cf073ad65289" />
