# Jenkins Freestyle Project: Build, Test, and Deploy a Banking Web App to Tomcat

A hands-on CI/CD project that uses a **Jenkins Freestyle job** to pull a **Java banking web application** from GitHub, build and test it with Maven, package it as a WAR file, and deploy it automatically to **Apache Tomcat** using the **Publish Over SSH** plugin.

> This is the banking-domain companion to the `ecommerce-app` project. If you already completed that lab, **reuse the same Jenkins and Tomcat servers** and jump straight to [Step 4](#step-4-create-the-jenkins-freestyle-job).

---

## Table of Contents

1. [About the Application](#1-about-the-application)
2. [Project Structure](#2-project-structure)
3. [Architecture](#3-architecture)
4. [Prerequisites](#4-prerequisites)
5. [Step 1: Create the EC2 Instances](#step-1-create-the-ec2-instances)
6. [Step 2: Set Up the Jenkins Server](#step-2-set-up-the-jenkins-server)
7. [Step 3: Set Up the Tomcat Server](#step-3-set-up-the-tomcat-server)
8. [Step 4: Create the Jenkins Freestyle Job](#step-4-create-the-jenkins-freestyle-job)
9. [Step 5: Add Build and Test Steps](#step-5-add-build-and-test-steps)
10. [Step 6: Configure Publish Over SSH](#step-6-configure-publish-over-ssh)
11. [Step 7: Add the Post-build Deploy Action](#step-7-add-the-post-build-deploy-action)
12. [Step 8: Run the Job and Verify](#step-8-run-the-job-and-verify)
13. [Troubleshooting](#troubleshooting)
14. [Key Learning Outcomes](#key-learning-outcomes)

---

## 1. About the Application

A simple in-memory banking application built with **Java Servlets + JSP + Maven** (packaged as a WAR).

| Feature | Details |
|---|---|
| Open account | Holder name and optional initial deposit |
| Deposit / Withdraw | Amount must be positive, max 2 decimal places; no overdraft |
| Fund transfer | Between two different accounts; atomic (nothing changes if it fails) |
| Statement | Per-account transaction history, newest first |
| Landing page | Responsive `index.html` with a hero illustration, feature cards and a CI/CD flow diagram |
| Unit tests | 13 JUnit 5 tests covering the business rules, executed by `mvn test` in Jenkins |

Two demo accounts (`ACC1001`, `ACC1002`) are created at startup. Data is kept in memory and resets when Tomcat restarts.

**Tech stack:** Java 17+ (built with JDK 21), Servlet API 4.0 (`javax.servlet`), JSP/JSTL 1.2, JUnit 5, Maven, Tomcat 9.

---

## 2. Project Structure

```text
banking-app/
├── pom.xml
├── README.md
└── src/
    ├── main/
    │   ├── java/com/bank/
    │   │   ├── model/      Account, Transaction, TransactionType
    │   │   ├── service/    BankService, BankException
    │   │   └── web/        BankServlet
    │   └── webapp/
    │       ├── index.html          Landing page (welcome file)
    │       ├── css/style.css       Styles for the dashboard pages
    │       ├── images/             Original SVG logo, hero illustration, icons
    │       └── WEB-INF/
    │           ├── web.xml
    │           └── views/  accounts.jsp, history.jsp
    └── test/java/com/bank/service/
        └── BankServiceTest.java
```

---

### Images and licensing

All images (`logo.svg`, `hero.svg`, `icon-*.svg`) were created from scratch for this project as plain SVG files, so there are **no third-party copyright or licence obligations**. You can edit them in any text editor. If you want to add photos, use sources that allow free use, such as Unsplash, Pexels or Pixabay, and check each image's licence.

---

## 3. Architecture

```text
Developer
    |
    | git push
    v
GitHub Repository
    |
    | Jenkins checks out source code
    v
Jenkins Freestyle Job
    |
    +--> Build:   mvn clean compile
    +--> Test:    mvn test
    +--> Package: mvn package
    |
    v
target/banking-app.war
    |
    | Publish Over SSH
    v
Tomcat Server (webapps/)
    |
    v
http://TOMCAT_IP:8080/banking-app/
```

---

## 4. Prerequisites

| Component | Purpose |
|---|---|
| Git / GitHub | Stores and downloads the source code |
| JDK 21 | Compiles and runs Java code |
| Jenkins | Automates build, test, and deploy |
| Maven | Builds and tests the application |
| Publish Over SSH (plugin) | Sends the WAR file to Tomcat |
| Apache Tomcat 9 | Runs the web application (`javax.servlet`) |
| Two Ubuntu EC2 instances | One for Jenkins, one for Tomcat |

> **Tomcat compatibility:** This app uses `javax.servlet`, so it needs **Tomcat 9**. It will not run on Tomcat 10+ without migrating to `jakarta.servlet`.

---

## Step 1: Create the EC2 Instances

1. Launch two Ubuntu EC2 instances: `jenkins-server` and `tomcat-server`.
2. Choose an instance type (for example `t2.medium` for Jenkins, `t2.micro` for Tomcat).
3. In the security groups, open **port 22** (SSH) and **port 8080** (Jenkins and Tomcat).

<!-- Add screenshots here -->

---

## Step 2: Set Up the Jenkins Server

SSH into the **Jenkins** instance.

```bash
# Update
sudo apt update -y

# Java
sudo apt install fontconfig openjdk-21-jre -y
java -version

# Jenkins
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y
sudo systemctl enable --now jenkins

# Maven
sudo apt install maven -y
mvn --version
```

Get the initial admin password, then open `http://JENKINS_PUBLIC_IP:8080`:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

> Maven needs the JDK to compile. If `mvn clean compile` reports "no compiler is provided", install the full JDK: `sudo apt install openjdk-21-jdk -y`.

---

## Step 3: Set Up the Tomcat Server

SSH into the **Tomcat** instance.

```bash
sudo apt update -y
sudo apt install openjdk-21-jre -y

wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.123/bin/apache-tomcat-9.0.123.tar.gz
tar -xvzf apache-tomcat-9.0.123.tar.gz

cd apache-tomcat-9.0.123/bin
./catalina.sh start
```

> If the download link no longer works, copy the current **tar.gz** link from the Tomcat 9 download page.

Open `http://TOMCAT_PUBLIC_IP:8080` and confirm the Tomcat welcome page appears.

---

## Step 4: Create the Jenkins Freestyle Job

1. Jenkins Dashboard → **New Item**.
2. Name: `Java-Banking-Build-Test-Deploy`.
3. Select **Freestyle project** → **OK**.
4. Under **Source Code Management** select **Git** and enter:

```text
https://github.com/YOUR_USERNAME/banking-app.git
```

5. Set the branch to `*/main` (add credentials if the repository is private).

---

## Step 5: Add Build and Test Steps

Add **two separate Execute shell** build steps so you can see which stage failed.

### 5.1 Build

```bash
set -e

echo "Starting Maven build..."
mvn clean compile

echo "Build completed successfully"
```

### 5.2 Test and package

```bash
set -e

echo "Running automated tests..."
mvn test

echo "Packaging WAR file..."
mvn package

echo "Checking generated WAR..."
ls -lh target/*.war
```

**What happens?**

- `mvn test` runs the 13 JUnit tests. If any fail, Jenkins stops **before** deployment.
- `mvn package` builds `target/banking-app.war` (the name comes from `<finalName>` in `pom.xml`).

---

## Step 6: Configure Publish Over SSH

### 6.1 Install the plugin

**Manage Jenkins → Plugins → Available plugins** → search **Publish Over SSH** → install. Restart Jenkins when no jobs are running.

### 6.2 Add the Tomcat server

**Manage Jenkins → System → Publish over SSH → SSH Servers → Add**

| Field | Value |
|---|---|
| Name | `tomcat-server` |
| Hostname | Tomcat server IP |
| Username | `ubuntu` |
| Remote Directory | Full path to Tomcat's `webapps` folder, e.g. `/home/ubuntu/apache-tomcat-9.0.123/webapps` |

Click **Advanced**, tick **Use password authentication, or use a different key**, and paste the Tomcat server's `.pem` private key. Click **Test Configuration**. You should see `Success`.

---

## Step 7: Add the Post-build Deploy Action

Open the job → **Configure → Post-build Actions → Send build artifacts over SSH**.

| Field | Value |
|---|---|
| SSH Server Name | `tomcat-server` |
| Source files | `target/*.war` |
| Remove prefix | `target` |
| Remote directory | *(leave empty)* |

Click **Save**.

---

## Step 8: Run the Job and Verify

1. Click **Build Now** and open **Console Output**. It should end with `Finished: SUCCESS`.
2. On the Tomcat server, check the deployment:

```bash
ls -lh apache-tomcat-9.0.123/webapps/
```

You should see `banking-app.war` and, after Tomcat unpacks it, a `banking-app/` folder.

3. If something looks wrong, check Tomcat:

```bash
ps -ef | grep tomcat
tail -n 100 apache-tomcat-9.0.123/logs/catalina.out
```

4. Open the application:

```text
http://TOMCAT_PUBLIC_IP:8080/banking-app/
```

The landing page (`index.html`) opens first. Click **Open App** or **Go to Dashboard** to reach the banking dashboard at `/banking-app/accounts`.

### Try it out

- Deposit into `ACC1001`, then check **History**.
- Withdraw more than the balance and see the error message.
- Transfer between `ACC1001` and `ACC1002`.
- Open a new account.

### Prove the pipeline gates deployment

Break a rule in `BankService` (for example, remove the overdraft check in `Account.debit`), push to GitHub, and run the job. `mvn test` fails and **nothing is deployed**. Fix it, push again, and it deploys.

---

## Troubleshooting

| Problem | What to check |
|---|---|
| `mvn: command not found` | Install Maven on the Jenkins server. |
| "No compiler is provided" | Install `openjdk-21-jdk` (not only the JRE). |
| Tests fail | Read the surefire output in the console log and fix the failing test. |
| `Permission denied (publickey)` | Check the pem key, username, and `authorized_keys` permissions. |
| SSH connection refused / timeout | Check port 22 in the Tomcat security group. |
| WAR file not found | Check `<packaging>war</packaging>` and `<finalName>` in `pom.xml`. |
| WAR not deployed | Check the Remote Directory path, folder permissions, and `catalina.out`. |
| Page loads without images | Check the `images/` folder is inside the WAR (`jar tf target/banking-app.war`). |
| HTTP 404 on `/banking-app/` | Wait a few seconds for unpacking; confirm the WAR name is `banking-app.war`. |
| JSP error about JSTL | JSTL must be packaged in the WAR (it is, via the `jstl` dependency). |
| Page errors on Tomcat 10+ | Use Tomcat 9, or migrate to `jakarta.servlet`. |

---

## Key Learning Outcomes

- Configure a Jenkins Freestyle project integrated with GitHub
- Build a Java web application with Maven
- Gate deployments on automated unit tests
- Package a WAR and deploy it to Tomcat with Publish Over SSH
- Read Jenkins console output and troubleshoot a failed pipeline
