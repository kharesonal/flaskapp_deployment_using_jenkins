# Jenkins CI/CD Pipeline for Flask Application

## Objective

This project aims to automate the testing and deployment of a simple Python Flask web application using a Jenkins pipeline.

## Project execution steps:

**Step1:Install Jenkins on your AWS VM**

```
sudo apt update
sudo apt upgrade -y
sudo apt install openjdk-11-jdk -y
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

```

* Access Jenkins: Once installed, open Jenkins in your browser using the server's IP address:

  `http://<server-ip>:8080`

* Get Jenkins first time passowrd for jenkins & user setup

  `sudo cat /var/lib/jenkins/secrets/initialAdminPassword`

## Step 2: Configuring Jenkins

  ### Install Required Plugins
  
  Navigate to Manage Jenkins → Manage Plugins.
  
  * Publish Over SSH
  * SSH Agent
  * Git plugin
  * Pipeline plugin
  * Email Extension plugin

## Step 3: Fork the python flask app repository

## Step 4: Create the Jenkins Pipeline

1 .Creating Credentials in Jenkins
  * Go to Jenkins Dashboard → Manage Jenkins → Manage Credentials.
  *  Add new SSH credentials for connecting to the deployment server:
     
    ID: EC2-SSH-Credentials
    Username: ubuntu
    Private Key: Paste the private key generated earlier.
    
2.Create a Jenkins Pipeline Job:

* In Jenkins, go to the Dashboard and click on New Item.

* Choose Pipeline and give the project a name "flaskapp"

* Under the Pipeline section, choose Pipeline script from SCM and select Git.

* Provide the URL of your forked Flask repository https://github.com/your-username/flask-app.git).](https://github.com/kharesonal/flaskapp_deployment_using_jenkins)

* Configure it


3.Create a Jenkinsfile in the repository: Add the following Jenkinsfile to the root of your repository:
```
pipeline {
    agent any

    environment {
        GIT_REPO = "https://github.com/kharesonal/flaskapp_deployment_using_jenkins.git"
        EC2_USER = 'ubuntu'
        EC2_HOST = '13.208.186.187'
        DEPLOY_DIR = '/home/ubuntu'
        SSH_CRED_ID = "EC2-SSH-Credentials"
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Cloning repository...'
                git branch: 'main', url: "${env.GIT_REPO}"
            }
        }

        stage('Build') {
            steps {
                echo 'Setting up Virtual Environment and Installing Dependencies...'
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    cd flaskapp
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Running Unit Tests...'
                sh '''
                    . venv/bin/activate
                    cd flaskapp
                    pytest test_app.py
                '''
            }
        }

        stage('Copy to EC2') {
            when {
                expression {
                    currentBuild.currentResult == 'SUCCESS'
                }
            }
            steps {
                sshagent (credentials: ["${env.SSH_CRED_ID}"]) {
                    echo 'Copying files to EC2 instance...'
                    sh '''
                        scp -o StrictHostKeyChecking=no -r * ${EC2_USER}@${EC2_HOST}:${DEPLOY_DIR}
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            when {
                expression {
                   currentBuild.result == null || currentBuild.result == 'SUCCESS'
                }
            }
            steps {
                sshagent (credentials: ["${env.SSH_CRED_ID}"]) {
                    echo 'Deploying to EC2...'
                    sh '''
                        ssh ${EC2_USER}@${EC2_HOST} << EOF
                            source venv/bin/activate
                            cd flaskapp
                            python3 app.py >> log.txt 2>&1 &
                        EOF
                    '''
                }
            }
        }
    }

      post {
        always {
            cleanWs()
            echo "Build completed with status: ${currentBuild.result}"
        }
        success {
            mail to: '89sonal.khare@gmail.com',
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "The build was successful!"
        }
        failure {
            mail to: '89sonal.khare@gmail.com',
                subject: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "The build failed. Please check Jenkins for more details."
        }
    }
    always {
            echo 'Pipeline finished.'
        }
}
```

3.Pipeline Breakdown:

* Build: This stage installs the project dependencies.
* Test: Executes unit tests using pytest.
* Deploy: If tests pass, the application is deployed (customize the deployment process as per your staging environment).
* Notifications:
   * On Success: Sends an email notification to the developer when the pipeline completes successfully.
   * On Failure: Sends an email notification to the developer when the build or any stage fails.
 
4. Set up a notification system to alert via email when the build process fails or succeeds.

Set up email notifications:

* Install the Email Extension Plugin in Jenkins.
* Configure the email notification settings by going to Manage Jenkins > Configure System > Extended E-mail Notification.
* Add your email server details (e.g., SMTP server, port, and credentials).
  
In the Jenkinsfile, the post section will handle email notifications on build failure or success.



