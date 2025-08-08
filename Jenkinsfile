pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-east-1'  // or your region
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    sh '''
                    python3 -m venv venv
                    ./venv/bin/pip install -r requirements.txt
                    ./venv/bin/pytest --junitxml=test-reports/results.xml || true
                    '''
                }
            }
            post {
                always {
                    script {
                        if (fileExists('test-reports/results.xml')) {
                            junit 'test-reports/results.xml'
                        } else {
                            echo 'No test reports found, skipping junit step.'
                        }
                    }
                }
            }
        }

        stage('Deploy Tomcat') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    sh 'ansible-playbook -i inventory/hosts.ini playbooks/deploy_tomcat.yml'
                }
            }
        }
    }
}
