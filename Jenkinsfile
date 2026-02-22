pipeline {
    agent any

    tools {
        jdk 'jdk'
        nodejs 'node17'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME = "subhadeep2001/starbucks:latest"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-token',
                    url: 'https://github.com/Aseemakram19/starbucks-kubernetes.git'
            }
        }

        stage('Sonarqube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=starbucks \
                        -Dsonar.projectKey=starbucks
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false,
                    credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {

                        sh "docker build -t starbucks ."
                        sh "docker tag starbucks ${IMAGE_NAME}"
                        sh "docker push ${IMAGE_NAME}"
                    }
                }
            }
        }

        stage('TRIVY IMAGE SCAN') {
            steps {
                sh "trivy image ${IMAGE_NAME} > trivyimage.txt"
            }
        }

        stage('DOCKER SCOUT SCAN') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker-scout quickview ${IMAGE_NAME}"
                        sh "docker-scout cves ${IMAGE_NAME}"
                        sh "docker-scout recommendations ${IMAGE_NAME}"
                    }
                }
            }
        }

        stage('App Deploy to Docker Container') {
            steps {
                sh '''
                    docker rm -f starbucks || true
                    docker run -d --name starbucks -p 3000:3000 ${IMAGE_NAME}
                '''
            }
        }
    }

    post {
        always {
            script {
                def buildStatus = currentBuild.currentResult
                def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId ?: 'Github User'

                emailext(
                    subject: "Pipeline ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <p>This is Jenkins Starbucks CI/CD pipeline status.</p>
                        <p><b>Project:</b> ${env.JOB_NAME}</p>
                        <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                        <p><b>Status:</b> ${buildStatus}</p>
                        <p><b>Started by:</b> ${buildUser}</p>
                        <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    """,
                    to: 'subhadeepghosh1270@gmail.com',
                    from: 'subhadeepghosh1270@gmail.com',
                    replyTo: 'subhadeepghosh1270@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
                )
            }
        }
    }
}
