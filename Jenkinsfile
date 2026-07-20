pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        timestamps()
    }

    triggers {
        pollSCM('* * * * *')
    }

    environment {
        EC2_HOST = credentials('ec2-host-ip')
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Downloading the latest code from GitHub...'
                checkout scm
            }
        }

        stage('Test') {
            steps {
                echo 'Testing index.html...'

                sh '''
                    test -s index.html
                    grep -qi '<html' index.html
                    echo "Website validation passed."
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo 'Deploying index.html to EC2...'

                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ec2-ssh-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        scp \
                          -i "${SSH_KEY}" \
                          -o StrictHostKeyChecking=accept-new \
                          index.html \
                          "${SSH_USER}@${EC2_HOST}:/tmp/jenkins-index.html"

                        ssh \
                          -i "${SSH_KEY}" \
                          -o StrictHostKeyChecking=accept-new \
                          "${SSH_USER}@${EC2_HOST}" '
                            set -e

                            sudo install -m 0644 \
                              /tmp/jenkins-index.html \
                              /var/www/html/index.html

                            rm /tmp/jenkins-index.html

                            sudo nginx -t
                            sudo systemctl reload nginx
                        '
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Website deployed successfully to EC2.'
        }

        failure {
            echo 'Pipeline failed. Check Console Output.'
        }
    }
}
