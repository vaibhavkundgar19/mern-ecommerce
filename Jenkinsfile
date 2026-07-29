pipeline {
    agent { label 'docker-agent' }

    environment {
        DOCKERHUB_USERNAME = "kundgar19"
        BACKEND_IMAGE      = "mern-backend"
        FRONTEND_IMAGE     = "mern-frontend"
        SCANNER_HOME       = tool 'sonar-scanner'
    }

    stages {

        stage('Set Branch Config') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        env.IMAGE_TAG     = "${BUILD_NUMBER}"
                        env.EXTRA_TAG     = "latest"
                        env.DEPLOY_ENV    = "prod"
                        env.K8S_NAMESPACE = "prod"
                        env.APP_DOMAIN    = "app.vaibhav.bond"
                    } else {
                        env.IMAGE_TAG     = "dev-${BUILD_NUMBER}"
                        env.EXTRA_TAG     = "dev-latest"
                        env.DEPLOY_ENV    = "dev"
                        env.K8S_NAMESPACE = "dev"
                        env.APP_DOMAIN    = "dev.vaibhav.bond"
                    }
                    env.APP_URL = "https://${env.APP_DOMAIN}"
                    echo "Branch=${env.BRANCH_NAME} | Env=${env.DEPLOY_ENV} | Tag=${env.IMAGE_TAG} | NS=${env.K8S_NAMESPACE} | URL=${env.APP_URL}"
                }
            }
        }

        stage('Cleanup & Checkout') {
            steps {
                cleanWs()
                checkout scm
                echo "Checkout complete - Branch: ${env.BRANCH_NAME} - Build #${BUILD_NUMBER}"
            }
        }

        stage('Install Dependencies') {
            parallel {
                stage('Backend Install') {
                    steps { dir('backend')  { sh 'npm install' } }
                }
                stage('Frontend Install') {
                    steps { dir('frontend') { sh 'npm install --legacy-peer-deps' } }
                }
            }
        }

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
                            trivy fs . --exit-code 0 --severity HIGH,CRITICAL \
                                --format table -o reports/trivy/fs-scan.txt
                            cat reports/trivy/fs-scan.txt
                        '''
                    }
                }
            }
        }

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

        stage('Build Docker Images') {
            steps {
                sh """
                    docker build \
                        -t ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG} \
                        -t ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${EXTRA_TAG} \
                        -f backend/Dockerfile ./backend
                """
                sh """
                    docker build \
                        -t ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        -t ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${EXTRA_TAG} \
                        -f frontend/Dockerfile ./frontend
                """
            }
        }

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

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    passwordVariable: 'DOCKER_PASS',
                    usernameVariable: 'DOCKER_USER'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh "docker push ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG}"
                    sh "docker push ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${EXTRA_TAG}"
                    sh "docker push ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG}"
                    sh "docker push ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${EXTRA_TAG}"
                    sh 'docker logout'
                }
            }
        }

        stage('Approval for Production') {
            when { branch 'main' }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Deploy to PRODUCTION (prod namespace)?", ok: "Deploy"
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-creds',
                                  accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                  secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {

                    sh 'aws eks update-kubeconfig --name mern-eks-cluster --region ap-south-1'

                    withCredentials([
                        string(credentialsId: 'MONGO_URI',      variable: 'MONGO_URI'),
                        string(credentialsId: 'SECRET_KEY',     variable: 'SECRET_KEY'),
                        string(credentialsId: 'EMAIL',          variable: 'APP_EMAIL'),
                        string(credentialsId: 'EMAIL_PASSWORD', variable: 'APP_PASSWORD')
                    ]) {
                        sh '''
                            kubectl create secret generic backend-secret \
                              --namespace=${K8S_NAMESPACE} \
                              --from-literal=MONGO_URI="$MONGO_URI" \
                              --from-literal=SECRET_KEY="$SECRET_KEY" \
                              --from-literal=EMAIL="$APP_EMAIL" \
                              --from-literal=PASSWORD="$APP_PASSWORD" \
                              --dry-run=client -o yaml | kubectl apply -f -
                        '''
                    }

                    sh 'kubectl apply -f k8s/ -n ${K8S_NAMESPACE}'

                    sh "kubectl set image deployment/mern-backend  backend=${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG}  -n ${K8S_NAMESPACE}"
                    sh "kubectl set image deployment/mern-frontend frontend=${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG} -n ${K8S_NAMESPACE}"

                    sh 'kubectl set env deployment/mern-backend ORIGIN="https://${APP_DOMAIN}" PRODUCTION="true" -n ${K8S_NAMESPACE}'

                    sh 'kubectl rollout status deployment/mern-backend  -n ${K8S_NAMESPACE} --timeout=180s'
                    sh 'kubectl rollout status deployment/mern-frontend -n ${K8S_NAMESPACE} --timeout=180s'
                }
            }
        }

        stage('Cleanup Old Images') {
            steps {
                sh 'docker image prune -f'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/**/*', allowEmptyArchive: true
            sh 'docker logout || true'
        }
        success {
            emailext(
                subject: "DEPLOYED: ${env.JOB_NAME} #${BUILD_NUMBER} (${env.BRANCH_NAME})",
                body: """<h3>Deployment Successful</h3>
                         <p>Branch: ${env.BRANCH_NAME}</p>
                         <p>Environment: ${env.DEPLOY_ENV}</p>
                         <p>Namespace: ${env.K8S_NAMESPACE}</p>
                         <p>Image Tag: ${env.IMAGE_TAG}</p>
                         <p>Live URL: <a href="${env.APP_URL}">${env.APP_URL}</a></p>
                         <p><a href="${BUILD_URL}">View Build</a></p>""",
                mimeType: 'text/html'
            )
        }
        failure {
            emailext(
                subject: "FAILED: ${env.JOB_NAME} #${BUILD_NUMBER} (${env.BRANCH_NAME})",
                body: """<h3>Build Failed</h3>
                         <p>Branch: ${env.BRANCH_NAME}</p>
                         <p><a href="${BUILD_URL}console">View Console Log</a></p>""",
                mimeType: 'text/html'
            )
        }
    }
}