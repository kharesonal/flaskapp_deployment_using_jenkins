pipeline {
    agent any

    environment {
        GIT_REPO = "https://github.com/kharesonal/flaskapp_deployment_using_jenkins.git"
        EC2_USER = 'ubuntu'
        EC2_HOST = '15.152.50.150'
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
                sshagent (credentials: ["${env.SSH_CRED_ID}"]) {
                    sh '''
                        ssh ${EC2_USER}@${EC2_HOST} << 'EOF'
                            cd flaskapp
                            source venv/bin/activate
                            pip install -r requirements.txt --break-system-packages
                        EOF
                    '''
                }
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
                    echo 'Deploying application on EC2...'
                    sh '''
                        ssh ${EC2_USER}@${EC2_HOST} << 'EOF'
                            cd ${DEPLOY_DIR}/flaskapp
                            . venv/bin/activate
                            nohup python app.py &
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
}
