pipeline {
    agent any

    environment {
        AWS_CREDENTIALS = credentials('aws-credentials')  // AWS creds from Jenkins
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-credentials', // Replace with your GitHub credential ID
                    url: 'https://github.com/creativesenthil/flask-app1.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'pip install -r requirements.txt'
            }
        }

        stage('Package App') {
            steps {
                sh 'tar -czf flask-app.tar.gz app/ wsgi.py requirements.txt'
            }
        }

        stage('Deploy via Ansible') {
            steps {
                ansiblePlaybook(
                    playbook: 'deploy.yml',
                    inventory: 'hosts.ini',
                    extras: "--extra-vars 'aws_access_key=${AWS_CREDENTIALS_USR} aws_secret_key=${AWS_CREDENTIALS_PSW}'"
                )
            }
        }
    }
}
