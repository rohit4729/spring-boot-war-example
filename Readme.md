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