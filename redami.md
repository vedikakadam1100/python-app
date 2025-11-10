# Python App – Fully Automated CI/CD Pipeline using Jenkins & AWS EC2

This project demonstrates how to set up a **Fully Automated Continuous Integration and Continuous Deployment (CI/CD) Pipeline** for a **Python web application** using **Jenkins**, **GitHub**, and **AWS EC2**.  

The pipeline automatically builds, deploys, and runs your Python app every time you push code to your GitHub repository .

---

##  Project Architecture

*GitHub → Jenkins Server → Deployment Server*

- **GitHub** stores your source code and triggers builds via a webhook.  
- **Jenkins Server** fetches the code, builds, and automates deployment.  
- **Deployment Server** runs the Python app automatically with **PM2**.




---

##  Infrastructure Overview

We use **two EC2 instances**:
| Instance | Purpose | Example Name | Example IP |
|-----------|----------|---------------|-------------|
|  **Jenkins Server** | Runs Jenkins and triggers deployment | `jenkins-server` | `172.31.8.5` |
|  **Deployment Server** | Hosts and runs the Python app | `deployment-server` | `172.31.x.x` |


---

##  Step-by-Step Setup Guide

### Step 1: Launch Two EC2 Instances
- Launch two **Ubuntu EC2 instances**:
  - One for **Jenkins**
  - One for **Deployment**
- Configure **security groups** to allow:
  - Port **22** (SSH)
  - Port **8080** (Jenkins)
  - Port **5000** or your app’s port


![](./img/Screenshot%202025-11-09%20215748.png)



---

###  Step 2: Configure Jenkins Server
1. **Connect to Jenkins EC2:**
   ```bash
   ssh -i "your-key.pem" ubuntu@<jenkins-server-ip>
Install Java and Jenkins:
```bash
# Update system packages
sudo apt update

# Install Java (Jenkins requires Java)
sudo apt install openjdk-17-jdk -y

# Add Jenkins repository key
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

# Add Jenkins repository to sources list
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update system again
sudo apt update

# Install Jenkins
sudo apt install jenkins -y

# Start Jenkins:
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins

# Access Jenkins at:
 http://<jenkins-server-ip>:8080

# Unlock Jenkins using:
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Complete setup → Install recommended plugins → Create admin user.
```
##  Step 3: Configure Deployment Server

### Connect to the Deployment EC2
```bash
ssh -i "your-key.pem" ubuntu@<deployment-server-ip>
```
### Install required packages:

```bash
sudo apt update
sudo apt install python3 python3-venv python3-pip npm -y
sudo npm install -g pm2
```
##  Step 4: Create GitHub Repository

### 1. Go to GitHub → Create a new repository (example: python-app-CICD)

### 2. Add all files:
```bash
app.py
requirements.txt
Dockerfile
Jenkinsfile
README.md
test
IMG
```
### 3.Commit and push:
```bash
git add .
git commit -m "Initial commit"
git push origin main
```

![](./img/Screenshot%202025-11-10%20161350.png)



##  Step 6: Add SSH Credentials in Jenkins

1. Go to **Jenkins → Manage Jenkins → Credentials → Global → Add Credentials**  
2. Select **SSH Username with private key**

###  Add the following details:
```bash
ID: node-app-key
Username: ubuntu
Private key: (paste from ~/.ssh/id_rsa)
```




##  Step 7: Create a New Pipeline Job

1. Open **Jenkins → New Item → Pipeline**  
2. Enter a job name (e.g., `Python-CICD`)  
3. Under **Pipeline Definition**, select **Pipeline script from SCM**  
   - Choose **Git**  
   - Paste your GitHub repository URL:  
     ```bash
     https://github.com/mansikadam1100/student-registration-CICD.git
     ```  
   - Set **Branch** to:  **main**

4. Click **Save**


![](./img/Screenshot%202025-11-10%20162313.png)



##  Step 8: Jenkinsfile (Pipeline Script)
```bash
pipeline {
    agent any

    environment {
        SSH_CRED = 'node-app-key'
        SERVER_IP = '52.66.247.199'
        REMOTE_USER = 'ubuntu'
        APP_DIR = '/home/ubuntu/python-app'
    }

    stages {
        stage('Clone Repo') {
            steps {
                git url: 'https://github.com/vedikakadam1100/python-app.git', branch: 'main'
            }
        }

        stage('Deploy to Server') {
            steps {
                sshagent(credentials: ["${SSH_CRED}"]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${SERVER_IP} "mkdir -p ${APP_DIR}"
                        scp -r Dockerfile README.md app.py requirements.txt test ${REMOTE_USER}@${SERVER_IP}:${APP_DIR}/
                    '''
                }
            }
        }

        stage('Install & Run App') {
            steps {
                sshagent(["${SSH_CRED}"]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${SERVER_IP} '
                            sudo apt update &&
                            sudo apt install -y python3-venv python3-pip npm &&
                            sudo npm install -g pm2 || true &&
                            cd ${APP_DIR} &&
                            python3 -m venv venv &&
                            source venv/bin/activate &&
                            pip install --upgrade pip &&
                            pip install -r requirements.txt &&
                            pm2 stop pythonapp || true &&
                            pm2 start app.py --name pythonapp --interpreter python3
                        '
                    """
                }
            }
        }
    }
}
```
##  Step 9: Configure GitHub Webhook

1.Open your GitHub repo **→ Settings → Webhooks → Add webhook**

2.Payload URL:
```bash
http://<jenkins-server-ip>:8080/github-webhook/
```
3.Content type: *application/json*

4.Trigger: **Just the push event**

5.Save webhook 


![](./img/Screenshot%202025-11-09%20214849.png)



##  Step 10: Run the Pipeline

1.Go to Jenkins → Your Pipeline → **Build Now**

2.Watch the logs → All stages (Clone, Deploy, Install & Run) should execute successfully.

3.After deployment, your Python app will be running on:
```bash
http://<deployment-server-ip>:5000
```

![](./img/Screenshot%202025-11-10%20162421.png)
![](./img/Screenshot%202025-11-10%20163242.png)



##  Pipeline Flow Summary

| **Stage** | **Description** |
|------------|------------------|
|  **Clone Repo** | Pulls the latest code from GitHub |
|  **Deploy to Server** | Copies project files to the Deployment EC2 instance |
|  **Install & Run App** | Installs dependencies and runs the app using PM2 |

##  Tools & Technologies

**Python 3**  
**Flask**  
**Jenkins**  
**AWS EC2**  
**PM2**  
**Git & GitHub**  
 **Ubuntu Linux**

## Project Structure
```bash
python-app-CICD/
│
├── app.py
├── requirements.txt
├── Dockerfile
├── Jenkinsfile
├── README.md
└── IMG/
```
##  Outcome

 **Fully automated CI/CD process**  
 **App auto-deploys on every GitHub push**  
 **Zero manual steps needed after setup**  
**Jenkins + PM2 keep your app running 24/7**


## Author

**mansi kadam**  
AWS | DevOps | Python Developer  
🔗 [GitHub Profile](https://github.com/vedikakadam1100)



---

##  Support

If you found this project helpful, please give it a  on **GitHub**!  
Sharing helps others learn **automation the smart way**  

A special thanks to **Trupti Ma’am** for her valuable guidance and support throughout this project 



## "Automation is not the future — it’s the present. Let Jenkins do the work for you!" 


 
