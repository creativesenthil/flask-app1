pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {
        // Stage 1: Code Commit (implied by Git SCM checkout)
        stage('Git Checkout') {
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
            }
        }

        // Stage 2: Build & Test
        stage('Build & Test') {
            steps {
                sh '''
                python -m venv venv
                ./venv/bin/pip install -r requirements.txt
                ./venv/bin/pytest  # Add your test command here
                '''
            }
            post {
                always {
                    junit '**/test-reports/*.xml'  # If you have test reports
                }
            }
        }

        // Stage 3: Initialize Ansible
        stage('Prepare Ansible') {
            steps {
                script {
                    env.COMMIT_ID = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.ARTIFACT_NAME = "flask-app-${env.COMMIT_ID}.tar.gz"
                    
                    sh """
                    tar -czf ${env.ARTIFACT_NAME} app/ wsgi.py requirements.txt templates/
                    """
                }
            }
        }

        // Stage 4: Deploy with Ansible
        stage('Ansible Deploy') {
            steps {
                script {
                    withCredentials([
                        sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY_FILE', usernameVariable: 'SSH_USER')
                    ]) {
                        sh """
                        ansible-playbook deploy.yml \
                            -i hosts.ini \
                            --private-key ${SSH_KEY_FILE} \
                            --extra-vars "\
                                app_port=5000 \
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
            cleanWs()
        }
        success {
            echo "Successfully deployed ${env.ARTIFACT_NAME} to Tomcat server"
        }
        failure {
            echo "Build failed! Check console output for details."
        }
    }
}
