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

### Step 2: Create and Configure the Build Job
We will create a specialized job to pull and test the source code:

1.  **New Item**: Create a new **Maven Project** named `helloWorld-test`.
2.  **Source Code Management**: 
    * Select **Git**.
    * **Repository URL**: [Insert your GitHub URL here]
    * **Branches to build**: `*/main` (or your preferred branch).
3.  **Build Section**:
    * **Root POM**: `pom.xml`
    * **Goals and options**: `test` 
    *(This ensures the code is validated before we move to the packaging stage.)*