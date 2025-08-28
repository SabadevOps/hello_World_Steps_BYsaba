Got it ✅
I’ll create a **complete README.md** file for you. This will combine **Jenkins + Tomcat + SonarQube setup** in a structured way so you can push it to GitHub and reuse anytime.

Here’s the full **README.md** 👇

---

````markdown
# 💙 Jenkins–Tomcat Deployment with SonarQube (CI/CD Setup) 💙

This guide explains how to set up a **CI/CD pipeline** with:
- Jenkins (on EC2)
- Apache Tomcat (on EC2)
- SonarQube (with embedded H2 DB)
- Maven (for build)

The pipeline automates **build, code quality scan, and deployment**.

---

## ⚙️ 1. Infrastructure Setup

- **EC2 #1 (Jenkins Server)**: `t2.large`, Ubuntu 20.04/22.04
- **EC2 #2 (Tomcat Server)**: normal instance, Ubuntu

---

## 🔧 2. Installations on Jenkins Server

1. **Install Java**
   ```bash
   sudo apt update
   sudo apt install openjdk-11-jdk -y
   java -version
````

2. **Install Maven**

   ```bash
   sudo apt install maven -y
   mvn -version
   ```

3. **Install Jenkins**

   ```bash
   curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
   /usr/share/keyrings/jenkins-keyring.asc > /dev/null
   echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
   https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
   /etc/apt/sources.list.d/jenkins.list > /dev/null
   sudo apt update
   sudo apt install jenkins -y
   ```

4. **Start Jenkins**

   ```bash
   sudo systemctl enable jenkins
   sudo systemctl start jenkins
   sudo systemctl status jenkins
   ```

5. **Access Jenkins**

   * URL: `http://<jenkins-ip>:8080`
   * Unlock using `/var/lib/jenkins/secrets/initialAdminPassword`

6. **Install Plugins**

   * Deploy to Container Plugin
   * Maven Integration Plugin
   * Eclipse Temurin JDK Plugin

7. **Configure Tools**

   * `Manage Jenkins → Global Tool Configuration`
   * Add JDK & Maven installation paths

---

## 🔧 3. Installations on Tomcat Server

1. **Install Java**

   ```bash
   sudo apt install openjdk-11-jdk -y
   ```

2. **Download & Install Tomcat**

   ```bash
   cd /opt
   wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.108/bin/apache-tomcat-9.0.108.tar.gz
   tar -xvzf apache-tomcat-9.0.108.tar.gz
   mv apache-tomcat-9.0.108 tomcat
   cd tomcat/bin
   ./startup.sh
   ```

3. **Access Tomcat**

   * URL: `http://<tomcat-ip>:8080`

---

## 🔧 4. Enable Remote Deployment in Tomcat

1. **Update Manager & Host-Manager configs**

   ```bash
   find / -name context.xml
   vi /opt/tomcat/webapps/manager/META-INF/context.xml
   vi /opt/tomcat/webapps/host-manager/META-INF/context.xml
   ```

   Comment out:

   ```xml
   <!--
   <Valve className="org.apache.catalina.valves.RemoteAddrValve"
          allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
   -->
   ```

2. **Add Users & Roles**

   ```bash
   vi /opt/tomcat/conf/tomcat-users.xml
   ```

   ```xml
   <role rolename="manager-gui"/>
   <role rolename="manager-script"/>
   <role rolename="manager-jmx"/>
   <role rolename="manager-status"/>
   <user username="deployer" password="deployer" roles="manager-script"/>
   ```

3. **Restart Tomcat**

   ```bash
   ./shutdown.sh
   ./startup.sh
   ```

---

## 🔧 5. Install & Configure SonarQube

1. **Install Java (if not installed)**

   ```bash
   sudo apt install openjdk-11-jdk -y
   ```

2. **Download & Extract SonarQube**

   ```bash
   cd /opt
   wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.5.1.90531.zip
   sudo apt install unzip -y
   unzip sonarqube-10.5.1.90531.zip
   mv sonarqube-10.5.1.90531 sonarqube
   ```

3. **Create SonarQube User**

   ```bash
   sudo adduser --system --no-create-home --group --disabled-login sonarqube
   sudo chown -R sonarqube:sonarqube /opt/sonarqube
   ```

4. **Create systemd Service**

   ```bash
   sudo vi /etc/systemd/system/sonarqube.service
   ```

   ```ini
   [Unit]
   Description=SonarQube service
   After=syslog.target network.target

   [Service]
   Type=forking
   ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
   ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
   User=sonarqube
   Group=sonarqube
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```

5. **Start SonarQube**

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable sonarqube
   sudo systemctl start sonarqube
   sudo systemctl status sonarqube
   ```

6. **Access SonarQube**

   * URL: `http://<server-ip>:9000`
   * Default login: `admin / admin`

7. **Generate Token**

   * Go to `My Account → Security → Generate Token`
   * Example:

     ```
     squ_4cb9951d19932c61abb01b040831a73aa3d4574b
     ```

---

## 🔑 6. Jenkins Credentials Setup

In **Jenkins → Manage Jenkins → Credentials → Global**:

* **Tomcat credentials:**
  ID = `tomcat-cred`
  Username = `deployer`
  Password = `deployer`

* **SonarQube token:**
  ID = `sonar-cred`
  Secret = `squ_4cb9951d19932c61abb01b040831a73aa3d4574b`

---

## 🔹 7. Freestyle Job Setup

1. **Create Freestyle Job**
2. Configure Git repo
3. **Build Step → Invoke Maven**

   ```
   clean install sonar:sonar -Dsonar.projectKey=my-project -Dsonar.host.url=http://52.206.34.13:9000 -Dsonar.login=squ_4cb9951d19932c61abb01b040831a73aa3d4574b
   ```
4. **Post-build Action → Deploy WAR/EAR to Container**

   * WAR file: `**/target/*.war`
   * Credentials: `tomcat-cred`
   * Tomcat URL: `http://<tomcat-ip>:8080/manager/text`

---

## 📜 8. Jenkins Pipeline (Alternative)

```groovy
pipeline {
    agent any

    tools {
        jdk 'java'
        maven 'maven'
    }

    environment {
        SONAR_URL = 'http://52.206.34.13:9000'
        SONAR_TOKEN = 'squ_4cb9951d19932c61abb01b040831a73aa3d4574b'
        TOMCAT_URL = 'http://52.206.34.13:8081/manager/text'
        TOMCAT_CRED = credentials('tomcat-cred')
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/SabadevOps/hello-world.git'
            }
        }

        stage('Build & SonarQube Analysis') {
            steps {
                sh '''
                    mvn clean install sonar:sonar \
                      -Dsonar.projectKey=my-project \
                      -Dsonar.host.url=$SONAR_URL \
                      -Dsonar.login=$SONAR_TOKEN
                '''
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                deploy adapters: [tomcat9(credentialsId: 'tomcat-cred', path: '', url: "$TOMCAT_URL")], 
                       contextPath: '/', war: '**/target/*.war'
            }
        }
    }
}
```

---

## 💛 Workflow Summary

1. Developer pushes code → GitHub
2. Jenkins pulls code
3. Maven builds project
4. SonarQube scan runs
5. WAR is deployed to Tomcat
6. Application is accessible 🚀

---

```

---

👉 This is a **ready-to-push README.md** for your GitHub repo.  

Do you also want me to **add architecture diagram (Jenkins → SonarQube → Tomcat)** in the README? It’ll look more professional.
```
