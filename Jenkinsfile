pipeline {
    agent any
    tools {
        jdk 'jdk17'
        gradle 'gradle7'  // Ensure you have configured Gradle in Jenkins
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        NEXUS_DOCKER_REPO = '192.168.80.142:5000'  // Nexus IP and Docker registry port
        IMAGE_NAME = 'adservice'
    }

    stages {
        stage('Git Checkout') {
            steps {
               git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/RaniaWachene1/Microservice.git'
            }
        }

        stage('Build') {
            steps {
                dir('adservice') {  // Navigate to the correct folder
                    sh "./gradlew build"
                }
            }
        }

        stage('Test') {
            steps {
                dir('adservice') {  // Ensure you are in the correct directory
                    sh "./gradlew test"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir('adservice') {  // Ensure we are in the correct directory
                    withSonarQubeEnv('sonar') {
                        sh '''
                            $SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectName=AddService \
                            -Dsonar.projectKey=AddService \
                            -Dsonar.sources=./src/main/java
                        '''
                    }
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                        // Running OWASP Dependency-Check with the NVD API key
                        dependencyCheck additionalArguments: '''
                            --scan ./ \
                            --format ALL \
                            --out . \
                            --prettyPrint \
                            --nvdApiKey ${NVD_API_KEY}
                        ''', odcInstallation: 'DC'

                        // Publish the Dependency-Check report
                        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                    }
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    // Build and tag the Docker image with the Nexus repository URL
                    sh "docker build -t ${NEXUS_DOCKER_REPO}/${IMAGE_NAME}:latest ."
                }
            }
        }

        stage('Login to Nexus Docker Registry') {
            steps {
                script {
                    // Use stored credentials to log into the Nexus Docker registry securely using --password-stdin
                    withCredentials([usernamePassword(credentialsId: 'nexus-docker-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "echo $PASS | docker login ${NEXUS_DOCKER_REPO} -u $USER --password-stdin"
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

        stage('Docker Image Scan') {
            steps {
                sh "trivy image --timeout 10m --scanners vuln --format table -o trivy-image-report.html ${NEXUS_DOCKER_REPO}/${IMAGE_NAME}:latest"
            }
        }
    }

    post {
        always {
            // Archive reports for all security scans and artifacts
            archiveArtifacts artifacts: '**/dependency-check-report.xml', allowEmptyArchive: true
            archiveArtifacts artifacts: '**/trivy-fs-report.html', allowEmptyArchive: true
            archiveArtifacts artifacts: '**/trivy-image-report.html', allowEmptyArchive: true
        }
    }
}
