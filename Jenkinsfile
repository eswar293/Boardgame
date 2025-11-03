pipeline {
    agent any
    tools {
        maven 'mvn-3.8'
        jdk 'jdk-11'
    }
    
    environment {
        SCANNER_HOME = tool 'sonarqube'
        IMAGE_NAME = "eswar1241/${env.JOB_NAME}"
        IMAGE_TAG = "v${env.BUILD_NUMBER}"
    }

    stages {
        stage('Clean WorkSpace') {
            steps {
                cleanWs()
            }
        }
        
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/eswar293/Boardgame.git'
            }
        }
        
        stage('Complie the Code') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Trivy File system Scan') {
            steps {
                sh 'trivy fs --format table -o filesystem-report.html .'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Broadgame \
                    -Dsonar.projectKey=Broadgame -Dsonar.java.binaries=. '''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-cred'
            }
        }
        
        stage('Build Application') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Deploy to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk-11', maven: 'mvn-3.8', mavenSettingsConfig: '',traceability: true) {
                    sh 'mvn deploy'
                }
            }
        }
        
        stage('Build Docker Image and Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', url: '') {
                        // echo "Removing old image ${IMAGE_NAME_TAG}" --- if any exists
                        sh "docker images ${IMAGE_NAME} --format '{{.Repository}}:{{.Tag}}' | grep -v ${IMAGE_TAG} | xargs -r docker rmi -f "
                        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                        
                    }
                }
            }
        }
        
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o image-report-html ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
        
        stage('Push Image to Docker Repo and run the image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', url: '') {
                        sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    }
                }
            }
        }
        
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'do-blr1-k8s', contextName: '', credentialsId: 'kube-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://3cc909a6-42b9-4a3d-8b14-26cd0f3a4d26.k8s.ondigitalocean.com') {
                    def BUILD_IMAGE_VERSION = "v${BUILD_NUMBER}"
                    sh """
                        sed -i 's|${IMAGE_TAG}|${BUILD_IMAGE_VERSION}|g' deployment-service.yaml
                        kubectl apply -f deployment-service.yaml
                    """
                    }
                }
            }
        }
        
        stage('Verify Application') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'do-blr1-k8s', contextName: '', credentialsId: 'kube-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://3cc909a6-42b9-4a3d-8b14-26cd0f3a4d26.k8s.ondigitalocean.com') {
                        sh "kubectl get pods -n webapps"
                        sh "kubectl get svc -n webapps"
                    }
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
        }
    }
}
}
