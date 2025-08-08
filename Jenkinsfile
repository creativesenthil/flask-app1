pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    environment {
        APP_PORT = '5000'
        EC2_IP = '23.22.202.254'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    extensions: [],
                    userRemoteConfigs: [[
                        credentialsId: 'github-credentials',
                        url: 'https://github.com/creativesenthil/flask-app1.git'
                    ]]
                ])
                script {
                    env.COMMIT_ID = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.ARTIFACT_NAME = "flask-app-${env.COMMIT_ID}.tar.gz"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                python -m venv venv
                ./venv/bin/pip install -r requirements.txt
                '''
            }
        }

        stage('Package App') {
            steps {
                sh """
                tar -czf ${env.ARTIFACT_NAME} app/ wsgi.py requirements.txt
                """
            }
        }

        stage('Deploy via Ansible') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY', passwordVariable: 'AWS_SECRET_KEY'),
                        sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY_FILE', usernameVariable: 'SSH_USER')
                    ]) {
                        sh """
                        ansible-playbook deploy.yml \
                            -i hosts.ini \
                            --private-key ${SSH_KEY_FILE} \
                            --extra-vars "\
                                aws_access_key=${AWS_ACCESS_KEY} \
                                aws_secret_key=${AWS_SECRET_KEY} \
                                ec2_ip=${EC2_IP} \
                                app_port=${APP_PORT} \
                                artifact_name=${env.ARTIFACT_NAME}
                            "
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                currentBuild.description = "Build ${env.BUILD_NUMBER} (${env.COMMIT_ID})"
                cleanWs()
            }
        }
        success {
            echo "Successfully deployed ${env.ARTIFACT_NAME} to ${EC2_IP}"
        }
        failure {
            echo "Build failed! Check console output for details."
        }
    }
}
