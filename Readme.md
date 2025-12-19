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
