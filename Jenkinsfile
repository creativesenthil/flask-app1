pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git 'https://your-repo-url.git'
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
                    inventory: 'hosts.ini'
                )
            }
        }
    }
}
