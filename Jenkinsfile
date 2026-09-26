pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node26'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'itsyogessh/swiggy-app'
    }

    stages {

        stage('Verify Tools') {
            steps {
                sh '''
                    java -version
                    node -v
                    npm -v
                    $SCANNER_HOME/bin/sonar-scanner --version
                    docker --version
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=Swiggy \
                        -Dsonar.projectName=Swiggy \
                        -Dsonar.sources=src \
                        -Dsonar.exclusions=node_modules/**,dist/**,dist-ssr/**
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build React App') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    trivy fs . \
                    --exit-code 0 \
                    --severity HIGH,CRITICAL \
                    -f table \
                    -o trivy-fs-report.txt
                '''

                archiveArtifacts(
                    artifacts: 'trivy-fs-report.txt',
                    allowEmptyArchive: true
                )
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                    -t itsyogessh/swiggy-app:${BUILD_NUMBER} \
                    -t itsyogessh/swiggy-app:latest .
                '''
            }
        }


        stage('DockerHub Push') {
            steps {
                withCredentials([
                    string(credentialsId: 'docker-token', variable: 'DOCKER_TOKEN')
                ]) {
                    sh '''
                        echo "$DOCKER_TOKEN" | docker login -u itsyogessh --password-stdin

                        docker push itsyogessh/swiggy-app:${BUILD_NUMBER}
                        docker push itsyogessh/swiggy-app:latest
                    '''
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image \
                    ${DOCKER_IMAGE}:latest \
                    --exit-code 0 \
                    --severity HIGH,CRITICAL \
                    -f table \
                    -o trivy-image-report.txt
                '''

                archiveArtifacts(
                    artifacts: 'trivy-image-report.txt',
                    allowEmptyArchive: true
                )
            }
        }

        stage('Deploy to Container') {
            steps {
                sh '''
                    docker rm -f swiggy || true
                    docker run -d \
                    --name swiggy \
                    -p 3000:3000 \
                    ${DOCKER_IMAGE}:latest
                '''
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution completed!'
        }

        failure {
            echo 'Pipeline failed. Check logs for details.'
        }

        success {
            echo 'Swiggy App deployed successfully!'
        }
    }
}
