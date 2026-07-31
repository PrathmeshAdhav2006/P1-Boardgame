pipeline {
    agent any
    tools {
        jdk 'jdk 11'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool "sonar"
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/PrathmeshAdhav2006/P1-Boardgame.git'
            }
        }
        stage('Compile') {
            steps {
                withMaven(jdk: 'jdk 11', maven: 'maven3', traceability: true) {
                    sh "mvn compile"
                }
            }
        }
        stage('Test') {
            steps {
                withMaven(jdk: 'jdk 11', maven: 'maven3', traceability: true) {
                    sh "mvn test"
                }
            }
        }
        stage('Scan File System') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("Sonar") {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BoardGame \
                        -Dsonar.projectKey=BoardGame \
                        -Dsonar.java.binaries=.
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-cred'
                }
            }
        }
        stage('Maven Build') {
            steps {
                withMaven(jdk: 'jdk 11', maven: 'maven3', traceability: true) {
                    sh "mvn package"
                }
            }
        }
        stage('Publish to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings-maven', jdk: 'jdk 11', maven: 'maven3', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }
        stage('Build and Tag Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'DockerHub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        sh "docker build -t prathmeshadhav2006/boardgame:latest ."
                    }
                }
            }
        }

        stage('Scan Docker Image') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html prathmeshadhav2006/boardgame:latest"
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'DockerHub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "docker push prathmeshadhav2006/boardgame:latest"
                    }
                }
            }
        }
        stage('Deploy to k8s') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'jenkins-svc-ac', namespace: 'webapp', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.41.37:6443') {
                    sh "kubectl apply -f k8s/deployment.yaml"
                    sh "kubectl apply -f k8s/service.yaml"
                }
            }
        }
        stage('Verify Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'jenkins-svc-ac', namespace: 'webapp', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.41.37:6443') {
                    sh "kubectl get pods -n webapp"
                    sh "kubectl get svc -n webapp"
                }
            }
        }
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
                    to: 'adhavprathmesh972@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}