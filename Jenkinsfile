pipeline {
    agent any;

    environment {
        DOCKERHUB_USERNAME  = "kundgar19"
        BACKEND_IMAGE       = "mern-backend"
        FRONTEND_IMAGE      = "mern-frontend"
        IMAGE_TAG           = "${BUILD_NUMBER}"
        SCANNER_HOME        = tool 'sonar-scanner'
        BACKEND_CONTAINER   = 'mern-backend'
        FRONTEND_CONTAINER  = 'mern-frontend'
        BACKEND_PORT        = '8000'
        EC2_PUBLIC_IP       = "13.126.203.252"
    }

    stages {

        // STAGE 1: CLEANUP & CHECKOUT .
        stage('Cleanup & Checkout') {
            steps {
                cleanWs()
                checkout scm
                echo "Code checkout complete - Build #${BUILD_NUMBER}"
            }
        }

        // STAGE 2: INSTALL DEPENDENCIES
        stage('Install Dependencies') {
            parallel {

                stage('Backend Install') {
                    steps {
                        dir('backend') {
                            sh 'npm install'
                        }
                    }
                }

                stage('Frontend Install') {
                    steps {
                        dir('frontend') {
                            sh 'npm install --legacy-peer-deps'
                        }
                    }
                }
            }
        }

        // STAGE 3: SECURITY SCANS
        stage('Security Scans') {
            parallel {

                // OWASP Dependency Check: Libraries mein vulnerabilities dhundhna
                stage('OWASP Dependency Check') {
                    steps {
                        sh 'mkdir -p reports/owasp'

                        dependencyCheck(
                            additionalArguments: '''
                                --scan backend/
                                --scan frontend/
                                --format HTML
                                --format XML
                                --out reports/owasp/
                                --disableAssembly
                                --disableYarnAudit
                                --disableNodeAudit
                                --prettyPrint
                            ''',
                            odcInstallation: 'DP-Check'
                        )

                        dependencyCheckPublisher(
                            pattern: 'reports/owasp/dependency-check-report.xml',
                            failedTotalCritical: 10,
                            unstableTotalCritical: 5
                        )
                    }
                }

                // Trivy FS Scan: Files aur secrets ko scan karna
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
                        -f backend/Dockerfile \
                        ./backend
                """

                echo "Building Frontend Image..."
                sh """
                    docker build \
                        --build-arg REACT_APP_BASE_URL=http://${EC2_PUBLIC_IP}:${BACKEND_PORT} \
                        -t ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        -t ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:latest \
                        -f frontend/Dockerfile \
                        ./frontend
                """
            }
        }

        // STAGE 6: TRIVY IMAGE SCAN
        stage('Trivy Image Scan') {
            steps {
                sh """
                    trivy image \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        -o reports/trivy/backend-image-scan.txt \
                        ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:latest

                    trivy image \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        -o reports/trivy/frontend-image-scan.txt \
                        ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:latest
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
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"

                        sh "docker push ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG}"
                        sh "docker push ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:latest"

                        sh "docker push ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG}"
                        sh "docker push ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:latest"

                        sh "docker logout"
                    }
                }
            }
        }

        // STAGE 8: DEPLOY
        stage('Deploy') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'MONGO_URI',        variable: 'MONGO_URI'),
                        string(credentialsId: 'SECRET_KEY',       variable: 'SECRET_KEY'),
                        string(credentialsId: 'EMAIL',            variable: 'EMAIL'),
                        string(credentialsId: 'EMAIL_PASSWORD',   variable: 'EMAIL_PASSWORD')
                    ]) {
                        echo "Stopping old containers..."
                        sh "docker stop ${BACKEND_CONTAINER}  || true"
                        sh "docker stop ${FRONTEND_CONTAINER} || true"
                        sh "docker rm   ${BACKEND_CONTAINER}  || true"
                        sh "docker rm   ${FRONTEND_CONTAINER} || true"

                        echo "Creating Docker network..."
                        sh "docker network create mern-network || true"

                        echo "Starting Backend Container..."
                        sh """
                            docker run -d \
                                --name ${BACKEND_CONTAINER} \
                                --network mern-network \
                                --restart unless-stopped \
                                -p ${BACKEND_PORT}:${BACKEND_PORT} \
                                -e MONGO_URI="${MONGO_URI}" \
                                -e ORIGIN="http://${EC2_PUBLIC_IP}" \
                                -e SECRET_KEY="${SECRET_KEY}" \
                                -e EMAIL="${EMAIL}" \
                                -e PASSWORD="${EMAIL_PASSWORD}" \
                                -e LOGIN_TOKEN_EXPIRATION="30d" \
                                -e OTP_EXPIRATION_TIME="120000" \
                                -e PASSWORD_RESET_TOKEN_EXPIRATION="2m" \
                                -e COOKIE_EXPIRATION_DAYS="30" \
                                -e PRODUCTION="true" \
                                -e NODE_ENV="production" \
                                ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG}
                        """

                        echo "Starting Frontend Container..."
                        sh """
                            docker run -d \
                                --name ${FRONTEND_CONTAINER} \
                                --network mern-network \
                                --restart unless-stopped \
                                -p 80:80 \
                                ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG}
                        """

                        echo "Waiting for containers to start..."
                        sh "sleep 15"

                        echo "Container Status"
                        sh "docker ps --filter 'name=${BACKEND_CONTAINER}'"
                        sh "docker ps --filter 'name=${FRONTEND_CONTAINER}'"

                        echo "Backend Health Check"
                        sh "curl -sf http://localhost:${BACKEND_PORT}/api/health && echo 'Backend Healthy' || echo 'Backend Health Failed'"

                        echo "Frontend Health Check"
                        sh "curl -sf http://localhost && echo 'Frontend Healthy' || echo 'Frontend Health Failed'"

                        echo "Application Live : http://${EC2_PUBLIC_IP}"
                    }
                }
            }
        }

        // STAGE 9: CLEANUP OLD IMAGES
        stage('Cleanup Old Images') {
            steps {
                echo "Cleaning dangling images..."
                sh "docker image prune -f"

                echo "Removing old backend images..."
                sh """
                    docker images ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE} --format "{{.Tag}}" \
                        | grep -v "latest" \
                        | grep -v "${IMAGE_TAG}" \
                        | xargs -r -I {} docker rmi ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:{} || true
                """

                echo "Removing old frontend images..."
                sh """
                    docker images ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE} --format "{{.Tag}}" \
                        | grep -v "latest" \
                        | grep -v "${IMAGE_TAG}" \
                        | xargs -r -I {} docker rmi ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:{} || true
                """
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/*/', allowEmptyArchive: true
            sh "docker logout || true"
        }

        success {
            echo "Build #${BUILD_NUMBER} deployed successfully - http://${EC2_PUBLIC_IP}"
        }

        failure {
            echo "Build #${BUILD_NUMBER} failed - ${BUILD_URL}console"
        }

        cleanup {
            cleanWs(
                cleanWhenSuccess: true,
                cleanWhenFailure: false,
                cleanWhenAborted: true
            )
        }
    }
}