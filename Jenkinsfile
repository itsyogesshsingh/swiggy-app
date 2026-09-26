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

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git(
                    branch: 'master',
                    credentialsId: 'github-swiggy-deploy',
                    url: 'git@github.com:itsyogessh/swiggy-app.git'
                )
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                    echo "===== JAVA ====="
                    java -version

                    echo "===== NODE ====="
                    node -v

                    echo "===== NPM ====="
                    npm -v

                    echo "===== SONAR SCANNER ====="
                    $SCANNER_HOME/bin/sonar-scanner --version

                    echo "===== DOCKER ====="
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
                        -Dsonar.sources=.
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    timeout(time: 2, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    npm install
                '''
            }
        }

        stage('Build React App') {
            steps {
                sh '''
                    npm run build
                '''
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
                    -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                    -t ${DOCKER_IMAGE}:latest \
                    .
                '''
            }
        }

        stage('DockerHub Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_TOKEN" | docker login \
                        -u "$DOCKER_USER" \
                        --password-stdin

                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest
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
            echo "Pipeline execution completed!"
        }

        failure {
            echo "Pipeline failed. Check logs for details."
        }

        success {
            echo "Swiggy App deployed successfully!"
        }
    }
}
