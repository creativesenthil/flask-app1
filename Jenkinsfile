pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        ansiColor('xterm')  // For colored console output
    }

    environment {
        // Core configuration
        EC2_IP = '23.22.202.254'
        APP_PORT = '5000'
        
        // Credentials from Jenkins vault
        AWS_CREDS = credentials('aws-credentials')
        SSH_KEY = credentials('ec2-ssh-key')
        VAULT_PASS = credentials('ansible-vault-pass')

        // Versioning
        COMMIT_ID = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        ARTIFACT_NAME = "flask-app-${COMMIT_ID}.tar.gz"
        BUILD_TAG = "build-${BUILD_NUMBER}-${COMMIT_ID}"
    }

    stages {
        stage('Checkout & Initialize') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    extensions: [
                        [$class: 'CleanBeforeCheckout'],
                        [$class: 'CloneOption', depth: 1, shallow: true]
                    ],
                    userRemoteConfigs: [[
                        credentialsId: 'github-credentials',
                        url: 'https://github.com/creativesenthil/flask-app1.git'
                    ]]
                ])
                
                // Verify critical files exist
                script {
                    def requiredFiles = ['app/', 'wsgi.py', 'requirements.txt', 'deploy.yml', 'hosts.ini']
                    for (file in requiredFiles) {
                        if (!fileExists(file)) {
                            error("Missing required file: ${file}")
                        }
                    }
                }
            }
        }

        stage('Setup Environment') {
            steps {
                sh '''
                python -m venv venv
                . venv/bin/activate
                pip install --upgrade pip==23.0.1
                pip install -r requirements.txt pytest coverage
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                . venv/bin/activate
                python -m pytest tests/ \
                    --junitxml=test-results.xml \
                    --cov=app \
                    --cov-report=xml \
                    --cov-report=html
                '''
            }
            post {
                always {
                    junit 'test-results.xml'
                    cobertura coberturaReportFile: 'coverage.xml'
                    archiveArtifacts artifacts: '**/coverage_html_report/**/*', allowEmptyArchive: true
                }
            }
        }

        stage('Build Artifact') {
            steps {
                sh """
                # Create versioned artifact
                tar -czvf ${ARTIFACT_NAME} \
                    --exclude='*.pyc' \
                    --exclude='__pycache__' \
                    app/ wsgi.py requirements.txt
                
                # Generate checksums
                md5sum ${ARTIFACT_NAME} > ${ARTIFACT_NAME}.md5
                sha256sum ${ARTIFACT_NAME} > ${ARTIFACT_NAME}.sha256
                """
                
                archiveArtifacts artifacts: "${ARTIFACT_NAME}*", fingerprint: true
            }
        }

        stage('Validate EC2 Connection') {
            steps {
                script {
                    try {
                        timeout(time: 2, unit: 'MINUTES') {
                            sh """
                            ssh -o StrictHostKeyChecking=no \
                                -o BatchMode=yes \
                                -i ${SSH_KEY} \
                                ubuntu@${EC2_IP} \
                                "echo 'EC2 connection successful'"
                            """
                        }
                    } catch (err) {
                        error("EC2 SSH connection failed. Verify:\n1. Security group rules\n2. SSH key permissions\n3. Instance status")
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            environment {
                ANSIBLE_CONFIG = "${WORKSPACE}/.ansible.cfg"
            }
            steps {
                script {
                    // Create temporary Ansible config
                    writeFile file: '.ansible.cfg', text: """
                    [defaults]
                    host_key_checking = False
                    retry_files_enabled = False
                    stdout_callback = yaml
                    """
                    
                    withCredentials([string(credentialsId: 'ansible-vault-pass', variable: 'VAULT')]) {
                        sh """
                        ansible-playbook deploy.yml \
                            -i hosts.ini \
                            --private-key ${SSH_KEY} \
                            --extra-vars "\
                                ec2_ip=${EC2_IP} \
                                app_port=${APP_PORT} \
                                artifact_name=${ARTIFACT_NAME} \
                                aws_access_key=${AWS_CREDS_USR} \
                                aws_secret_key=${AWS_CREDS_PSW} \
                                build_tag=${BUILD_TAG} \
                            " \
                            --vault-password-file <(echo \$VAULT) \
                            -v
                        """
                    }
                }
            }
        }

        stage('Smoke Test') {
            steps {
                retry(3) {
                    timeout(time: 5, unit: 'MINUTES') {
                        script {
                            def response = httpRequest(
                                url: "http://${EC2_IP}:${APP_PORT}/health",
                                validResponseCodes: '200,403,500',
                                timeout: 30
                            )
                            
                            if (response.status != 200 || !response.content.contains('"status":"ok"')) {
                                error("Smoke test failed. Response: ${response.status} - ${response.content}")
                            }
                            
                            echo "Deployment verified: ${response.content}"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // Clean sensitive files
                sh 'rm -f .ansible.cfg || true'
                
                // Update build description
                currentBuild.description = "v${BUILD_NUMBER} (${COMMIT_ID}) to ${EC2_IP}"
                
                // Clean workspace (keep artifacts)
                cleanWs(cleanWhenFailure: false)
            }
        }
        
        success {
            slackSend(
                color: 'good',
                message: """✅ Success: ${JOB_NAME} #${BUILD_NUMBER}
                | Commit: ${COMMIT_ID}
                | EC2: ${EC2_IP}:${APP_PORT}
                | Health: http://${EC2_IP}:${APP_PORT}/health
                | Console: ${BUILD_URL}"""
            )
        }
        
        failure {
            slackSend(
                color: 'danger',
                message: """❌ Failed: ${JOB_NAME} #${BUILD_NUMBER}
                | Commit: ${COMMIT_ID}
                | EC2: ${EC2_IP}
                | Console: ${BUILD_URL}"""
            )
            
            // Optional: Trigger rollback playbook
            sh """
            ansible-playbook rollback.yml \
                -i hosts.ini \
                --private-key ${SSH_KEY} \
                --extra-vars "ec2_ip=${EC2_IP}" \
                --tags rollback
            """
        }
    }
}
