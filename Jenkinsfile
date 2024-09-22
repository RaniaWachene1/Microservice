pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        NEXUS_DOCKER_REPO = '192.168.80.142:5000'  // Nexus IP and Docker registry port
        IMAGE_NAME = 'adservice'
    }
    
    stages {
        // Add a secrets scanning stage here

        // Add SAST tool like Checkmarx or Semgrep for deeper code security analysis

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
        script {
            def jobName = env.JOB_NAME
            def buildNumber = env.BUILD_NUMBER
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

            def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
            """

            emailext (
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                body: body,
                to: 'rania.wachene@esprit.tn',
                from: 'raniawachen21@gmail.com',
                replyTo: 'rania.wachene@esprit.tn',
                mimeType: 'text/html',
                attachmentsPattern: '**/dependency-check-report.xml,**/trivy-fs-report.html,**/trivy-image-report.html'

            )
        }
        archiveArtifacts artifacts: '**/dependency-check-report.xml', allowEmptyArchive: true
        archiveArtifacts artifacts: '**/trivy-fs-report.html', allowEmptyArchive: true
        archiveArtifacts artifacts: '**/trivy-image-report.html', allowEmptyArchive: true
    }
}


}
