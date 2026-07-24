pipeline {
    agent { label 'docker-agent' }

    environment {
        DOCKERHUB_USERNAME = "kundgar19" 
        BACKEND_IMAGE      = "mern-backend"
        FRONTEND_IMAGE     = "mern-frontend"
        IMAGE_TAG          = "${BUILD_NUMBER}"
        SCANNER_HOME       = tool 'sonar-scanner'
    }

    stages {

        // STAGE 1: CLEANUP & CHECKOUT .
        stage('Cleanup & Checkout') {
            steps {
                cleanWs()
                checkout scm
                echo "Code checkout complete - Branch: ${env.BRANCH_NAME} - Build #${BUILD_NUMBER}"
            }
        }

        // STAGE 2: INSTALL DEPENDENCIES
        stage('Install Dependencies') {
            parallel {
                stage('Backend Install') {
                    steps {
                        dir('backend') { sh 'npm install' }
                    }
                }
                stage('Frontend Install') {
                    steps {
                        dir('frontend') { sh 'npm install --legacy-peer-deps' }
                    }
                }
            }
        }

        // STAGE 3: SECURITY SCANS
        stage('Security Scans') {
            parallel {

                stage('OWASP Dependency Check') {
                    steps {
                        sh 'mkdir -p reports/owasp'
                        withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_KEY')]) {
                            dependencyCheck(
                                additionalArguments: """
                                    --scan backend/
                                    --scan frontend/
                                    --format HTML
                                    --format XML
                                    --out reports/owasp/
                                    --nvdApiKey ${NVD_KEY}
                                    --disableAssembly
                                    --disableYarnAudit
                                    --disableNodeAudit
                                    --prettyPrint
                                """,
                                odcInstallation: 'DP-Check'
                            )
                        }
                        dependencyCheckPublisher(
                            pattern: 'reports/owasp/dependency-check-report.xml',
                            failedTotalCritical: 10,
                            unstableTotalCritical: 5
                        )
                    }
                }

                stage('Trivy FS Scan') {
                    steps {
                        sh '''
                            mkdir -p reports/trivy
                            trivy fs . \
                                --exit-code 0 \
                                --severity HIGH,CRITICAL \
                                --format table \
                                -o reports/trivy/fs-scan.txt
                            cat reports/trivy/fs-scan.txt
                        '''
                    }
                }
            }
        }

        // STAGE 4: SONARQUBE ANALYSIS
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=mern-ecommerce \
                        -Dsonar.projectName=mern-ecommerce \
                        -Dsonar.sources=backend,frontend
                    """
                }
            }
        }

        // STAGE 5: BUILD DOCKER IMAGES
        stage('Build Docker Images') {
            steps {
                echo "Building Backend Image..."
                sh """
                    docker build \
                        -t ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG} \
                        -t ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:latest \
                        -f backend/Dockerfile ./backend
                """

                echo "Building Frontend Image..."
                sh """
                    docker build \
                        -t ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        -t ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:latest \
                        -f frontend/Dockerfile ./frontend
                """
            }
        }

        // STAGE 6: TRIVY IMAGE SCAN
        stage('Trivy Image Scan') {
            steps {
                sh """
                    mkdir -p reports/trivy

                    trivy image --exit-code 0 --severity HIGH,CRITICAL \
                        --format table -o reports/trivy/backend-image-scan.txt \
                        ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG}

                    trivy image --exit-code 0 --severity HIGH,CRITICAL \
                        --format table -o reports/trivy/frontend-image-scan.txt \
                        ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG}
                """
            }
        }

        // STAGE 7: PUSH TO DOCKER HUB
        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-creds',
                        passwordVariable: 'DOCKER_PASS',
                        usernameVariable: 'DOCKER_USER'
                    )]) {
                        sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'

                        sh "docker push ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG}"
                        sh "docker push ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:latest"
                        sh "docker push ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG}"
                        sh "docker push ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:latest"

                        sh "docker logout"
                    }
                }
            }
        }

        // STAGE 8: CLEANUP OLD IMAGES
        stage('Cleanup Old Images') {
            steps {
                sh "docker image prune -f"
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/**/*', allowEmptyArchive: true
            sh "docker logout || true"
        }
        success {
            emailext(
                subject: "SUCCESS: ${env.JOB_NAME} - Build #${BUILD_NUMBER}",
                body: """<p>Build successful!</p>
                         <p>Branch: ${env.BRANCH_NAME}</p>
                         <p>Images pushed with tag: ${IMAGE_TAG}</p>
                         <p><a href="${BUILD_URL}">View Build</a></p>""",
                mimeType: 'text/html'
            )
        }
        failure {
            emailext(
                subject: "FAILED: ${env.JOB_NAME} - Build #${BUILD_NUMBER}",
                body: """<p>Build failed.</p>
                         <p>Branch: ${env.BRANCH_NAME}</p>
                         <p><a href="${BUILD_URL}console">View Console Log</a></p>""",
                mimeType: 'text/html'
            )
        }
    }
}