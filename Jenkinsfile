pipeline {
    agent any
     environment {
        SCANNER_HOME= tool 'sonar-scanner'
        NEXUS_DOCKER_REPO = '192.168.80.142:5000'  // Nexus IP and Docker registry port
        IMAGE_NAME = 'myapp'
    }
    
    stages {
         stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SonarQube Analsyis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=AddService -Dsonar.projectKey=AddService \
                            -Dsonar.java.binaries=. '''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                  waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            }
        }
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t raniawachene/adservice:latest ."
                    }
                }
            }
        }
        stage('Login to Nexus Docker Registry') {
            steps {
                script {
                    // Use stored credentials to log into the Nexus Docker registry
                    withCredentials([usernamePassword(credentialsId: 'nexus-docker-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "docker login ${NEXUS_DOCKER_REPO} -u $USER -p $PASS"
                    }
                }
            }
        }

        stage('Push Docker Image to Nexus') {
            steps {
                script {
                    // Push the Docker image to the Nexus Docker registry
                    sh "docker push ${NEXUS_DOCKER_REPO}/${IMAGE_NAME}:latest"
                }
            }
        }
    }
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push raniawachene/adservice:latest "
                    }
                }
            }
        }
    }
}
