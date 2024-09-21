pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        NEXUS_DOCKER_REPO = '192.168.80.142:5000'  // Nexus IP and Docker registry port
        IMAGE_NAME = 'email-service'
    }
    
    stages {
   
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=AddService \
                        -Dsonar.projectKey=AddService \
                        -Dsonar.java.binaries=.
                    '''
                }
            }
        }

        // Add infrastructure as code (IaC) scan if using Terraform, etc.
        
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                        dependencyCheck additionalArguments: '''
                            --scan ./ \
                            --format ALL \
                            --out . \
                            --prettyPrint \
                            --nvdApiKey ${NVD_API_KEY}
                        ''', odcInstallation: 'DC'

                        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                    }
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    sh "docker build -t ${NEXUS_DOCKER_REPO}/${IMAGE_NAME}:latest ."
                }
            }
        }

        // Add integration testing stage here

        stage('Login to Nexus Docker Registry') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'nexus-docker-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "echo $PASS | docker login ${NEXUS_DOCKER_REPO} -u $USER --password-stdin"
                    }
                }
            }
        }

        stage('Push Docker Image to Nexus') {
            steps {
                script {
                    sh "docker push ${NEXUS_DOCKER_REPO}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh "trivy image --timeout 10m --scanners vuln --format table -o trivy-image-report.html ${NEXUS_DOCKER_REPO}/${IMAGE_NAME}:latest"
            }
        }

        // Add dynamic application security testing (DAST) using OWASP ZAP for runtime security tests

        // Add Canary or Blue-Green Deployment if needed
    }

    post {
        always {
            archiveArtifacts artifacts: '**/dependency-check-report.xml', allowEmptyArchive: true
            archiveArtifacts artifacts: '**/trivy-fs-report.html', allowEmptyArchive: true
            archiveArtifacts artifacts: '**/trivy-image-report.html', allowEmptyArchive: true
        }
    }
}
