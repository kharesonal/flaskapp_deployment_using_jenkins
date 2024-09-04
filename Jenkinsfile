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
            mail to: '89sonal.khare@gmail.com',
                 subject: "Jenkins Build: ${currentBuild.fullDisplayName}",
                 body: "Build ${currentBuild.result}: ${currentBuild.fullDisplayName}\nMore info at: ${env.BUILD_URL}"
        }
    }
}
