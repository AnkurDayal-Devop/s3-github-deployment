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
        AWS_DEFAULT_REGION = 'eu-north-1'
        DEPLOYMENT_BUCKET  = 'webapp-releases-ec2'
        EC2_INSTANCE_ID    = 'i-06654ce7d5adbd1cd'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -e

                    test -s index.html
                    grep -qi '<html' index.html

                    echo "Website validation passed."
                '''
            }
        }

        stage('Package') {
            steps {
                script {
                    env.REVISION = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    if (!env.REVISION) {
                        error('Unable to determine Git revision.')
                    }

                    echo "Release version: ${env.REVISION}"
                }

                sh '''
                    set -e

                    rm -f release.zip
                    zip -qr release.zip index.html

                    test -s release.zip
                    unzip -l release.zip
                '''
            }
        }

        stage('Publish Release') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-jenkins']
                ]) {
                    sh '''
                        set -e

                        aws s3 cp release.zip \
                          "s3://${DEPLOYMENT_BUCKET}/releases/webapp-${REVISION}.zip"

                        printf '%s' "${REVISION}" > current-version.txt

                        aws s3 cp current-version.txt \
                          "s3://${DEPLOYMENT_BUCKET}/current-version.txt"
                    '''
                }
            }
        }

        stage('Deploy with SSM') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-jenkins']
                ]) {
                    script {
                        env.SSM_COMMAND_ID = sh(
                            script: '''
                                aws ssm send-command \
                                  --document-name "AWS-RunShellScript" \
                                  --instance-ids "$EC2_INSTANCE_ID" \
                                  --parameters "commands=[\\"sudo /usr/local/bin/deploy-webapp ${REVISION}\\"]" \
                                  --comment "Deploy release ${REVISION}" \
                                  --query 'Command.CommandId' \
                                  --output text
                            ''',
                            returnStdout: true
                        ).trim()

                        if (!env.SSM_COMMAND_ID ||
                            env.SSM_COMMAND_ID == 'None') {
                            error('SSM did not return a valid command ID.')
                        }

                        echo "SSM Command ID: ${env.SSM_COMMAND_ID}"
                    }
                }
            }
        }

        stage('Wait and Check Deployment') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-jenkins']
                ]) {
                    script {
                        int waitStatus = sh(
                            script: '''
                                aws ssm wait command-executed \
                                  --command-id "$SSM_COMMAND_ID" \
                                  --instance-id "$EC2_INSTANCE_ID"
                            ''',
                            returnStatus: true
                        )

                        sh '''
                            echo "SSM deployment result:"

                            aws ssm get-command-invocation \
                              --command-id "$SSM_COMMAND_ID" \
                              --instance-id "$EC2_INSTANCE_ID" \
                              --query '{
                                  Status:Status,
                                  ExitCode:ResponseCode,
                                  Output:StandardOutputContent,
                                  Error:StandardErrorContent
                              }'
                        '''

                        if (waitStatus != 0) {
                            error(
                                'Deployment failed on EC2. ' +
                                'Review the SSM output above.'
                            )
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Release ${REVISION} deployed successfully."
        }

        failure {
            echo "Pipeline failed. Check the failed stage and SSM output."
        }

        always {
            sh 'rm -f release.zip current-version.txt'
        }
    }
}
