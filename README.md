# Atom Project - CI Pipeline

A fully automated CI pipeline for a java-based web application using **Jenkins**, **SonarQube**, **Docker**, and **AWS ECR** that triggers on every push to the `main` branch.

---

## Tech Stack

| Category         | Tool / Service        |
| ---------------- | --------------------- |
| Language         | Java (JDK 21)         |
| Build            | Maven 3.9             |
| Code Analysis    | SonarQube, Checkstyle |
| Containerization | Docker                |
| Registry         | AWS ECR               |
| CI               | Jenkins               |

---

## Pipeline Overview

The Jenkinsfile defines the following stages:

```
Fetch Code → Build → Unit Test → Checkstyle Analysis → SonarQube Code Analysis → Build Docker Image → Push to ECR → Cleanup
```

| Stage                       | Description                                                    |
| --------------------------- | -------------------------------------------------------------- |
| **Fetch Code**              | Clones the `main` branch from GitHub                           |
| **Build**                   | Runs `mvn clean install -DskipTests`, archives the `.war` file |
| **Unit Test**               | Runs `mvn test`                                                |
| **Checkstyle Analysis**     | Runs `mvn checkstyle:checkstyle` for code style checks         |
| **SonarQube Code Analysis** | Performs static analysis via SonarQube Scanner                 |
| **Build App Image**         | Builds Docker image tagged with `BUILD_NUMBER`                 |
| **Upload Image to ECR**     | Pushes image to AWS ECR (`latest` + build-numbered tag)        |
| **Remove Container**        | Cleans up local Docker images                                  |

---

## Jenkins Setup Guide

### Prerequisites

- An **EC2 instance** (Ubuntu recommended) with Jenkins, Docker, and JDK 21 installed.
- A **SonarQube server** running and accessible (e.g., on a separate EC2 or container at port `80`/`9000`).
- An **AWS account** with ECR repository created.

---

### 1. AWS IAM User

Created an IAM user named **`jenkins`** with the following:

- **Permissions**: Attach **`AmazonEC2ContainerRegistryFullAccess`** policy (or equivalent ECR full-access policy).
- **Access Keys**: Generate an **Access Key ID** and **Secret Access Key** (under _Security credentials → Access keys_). Need them for Jenkins credentials.

---

### 2. Jenkins Plugins

Navigated to **Settings → Manage Jenkins → Plugins → Available Plugins** and installed the following:

| Plugin                                 | Purpose                                      |
| -------------------------------------- | -------------------------------------------- |
| Git                                    | Source code checkout (usually pre-installed) |
| JDK Tool                               | JDK management (usually pre-installed)       |
| Maven Integration                      | Maven build support (usually pre-installed)  |
| **Build Timestamp**                    | Adds build timestamp variable to builds      |
| **SonarQube Scanner**                  | SonarQube integration for code analysis      |
| **Pipeline Maven Integration**         | Maven support in pipeline scripts            |
| **Slack Notification**                 | Sends build notifications to Slack           |
| **Amazon Web Services SDK :: All**     | AWS SDK for Jenkins                          |
| **Docker Pipeline**                    | Docker support in pipeline scripts           |
| **Amazon ECR**                         | ECR authentication and push/pull support     |
| **CloudBees Docker Build and Publish** | Docker build & publish from Jenkins          |

> **Note**: Restart Jenkins after installing all plugins.

---

### 3. Tool Configuration

Navigate to **Settings → Manage Jenkins → Tools**.

#### Maven

| Field                 | Value      |
| --------------------- | ---------- |
| Name                  | `MAVEN3.9` |
| Install automatically | ✅ Checked |
| Version               | `3.9.x`    |

#### JDK

> JDK 17/21 must be **manually installed** on the EC2 instance first (e.g., `sudo apt install openjdk-21-jdk`).

| Field                 | Value                                                      |
| --------------------- | ---------------------------------------------------------- |
| Name                  | `JDK17`                                                    |
| JAVA_HOME             | `/usr/lib/jvm/java-21-openjdk-amd64` (or your actual path) |
| Install automatically | ❌ Unchecked                                               |

#### SonarQube Scanner

| Field                 | Value        |
| --------------------- | ------------ |
| Name                  | `SONAR6.2`   |
| Install automatically | ✅ Checked   |
| Version               | `6.2.1.4610` |

---

### 4. System Configuration

Navigate to **Settings → Manage Jenkins → System**.

#### Build Timestamp

| Field    | Value                                          |
| -------- | ---------------------------------------------- |
| Pattern  | `ddMMYY-HHmm`                                  |
| Timezone | _(Select your timezone, e.g., `Asia/Kolkata`)_ |

#### SonarQube Server

1. ✅ Check **"Environment variables"** checkbox.
2. Click **"Add SonarQube"**.

| Field                       | Value                              |
| --------------------------- | ---------------------------------- |
| Name                        | `SONAR-SERVER`                     |
| Server URL                  | `http://<sonarqube-private-ip>:80` |
| Server authentication token | _(Add as credential — see below)_  |

**Generate SonarQube Token:**

1. Log into SonarQube UI → Click your avatar → **My Account** → **Security**.
2. Under _Generate Tokens_, give it a name (e.g., `jenkins`) and click **Generate**.
3. Copy the token.

**Add Token as Jenkins Credential:**

1. In the Server authentication token dropdown, click **Add → Jenkins**.
2. Kind: **Secret text**
3. Secret: _(paste the SonarQube token)_
4. ID: `sonartoken` _(or any descriptive ID)_
5. Click **Add**, then select the newly created credential.

#### AWS Credentials

1. Navigate to **Settings → Manage Jenkins → Credentials → System → Global credentials**.
2. Click **"Add Credentials"**.

| Field             | Value                       |
| ----------------- | --------------------------- |
| Kind              | **AWS Credentials**         |
| ID                | `jenkinsecrcreds`           |
| Description       | `Jenkins ECR Access`        |
| Access Key ID     | _(from IAM user `jenkins`)_ |
| Secret Access Key | _(from IAM user `jenkins`)_ |

---

### 5. Pipeline Job Setup

1. **Dashboard → New Item** → Enter name (e.g., `atom-project-ci`) → Select **Pipeline** → OK.
2. Under **Pipeline**:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/Shadab909/atom-project-app.git`
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
3. Click **Save**.
4. Click **Build Now** to trigger the pipeline.

---

### 6. Environment Variables in Jenkinsfile

These are the key variables defined in the `Jenkinsfile` — update them to match your setup:

```groovy
environment {
    sonarScanner     = tool 'SONAR6.2'              // Must match SonarQube Scanner tool name
    registryCredential = 'ecr:us-east-1:jenkinsecrcreds'  // Must match AWS credential ID
    imageName        = "596517178703.dkr.ecr.us-east-1.amazonaws.com/atomecrrepo"
    ecrRegistry      = "https://596517178703.dkr.ecr.us-east-1.amazonaws.com"
}

tools {
    maven "MAVEN3.9"   // Must match Maven tool name
    jdk   "JDK17"      // Must match JDK tool name
}
```

> **Important**: Ensure the tool names and credential IDs in the Jenkinsfile exactly match what you configured in Jenkins.

---

## Docker

The application is packaged as a Docker image using `tomcat:10-jdk21`:

```dockerfile
FROM tomcat:10-jdk21
RUN rm -rf /usr/local/tomcat/webapps/*
COPY target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

---

## Local Development

```bash
# Build
mvn clean install

# Run tests
mvn test

# Checkstyle
mvn checkstyle:checkstyle

# Docker build (after Maven build)
docker build -t atom-project-app .
docker run -d -p 8080:8080 atom-project-app
```
