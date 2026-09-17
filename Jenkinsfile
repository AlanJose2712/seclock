pipeline {

    agent any

    environment {
        AWS_REGION = 'ap-southeast-2'
        AWS_ACCOUNT_ID = '394691794638'

        ECR_REPOSITORY = 'seclock'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME = "${ECR_REGISTRY}/${ECR_REPOSITORY}"

        SONAR_PROJECT_KEY = 'seclock'
        SONAR_PROJECT_NAME = 'Seclock'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh '''
                    echo "Running Seclock tests..."

                    docker build -t seclock-test:${BUILD_NUMBER} .

                    docker run --rm \
                        seclock-test:${BUILD_NUMBER} \
                        python test_e2e.py
                '''
            }
        }

             stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonarscanner'
                    withSonarQubeEnv('sonarqube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                -Dsonar.projectName="${SONAR_PROJECT_NAME}" \
                                -Dsonar.sources=. \
                                -Dsonar.tests=test_e2e.py \
                                -Dsonar.exclusions="__pycache__/**,.venv/**,venv/**,sample_certificates/**"
                        """
                    }
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "Building Seclock Docker image..."

                    docker build \
                        -t ${IMAGE_NAME}:${GIT_COMMIT} \
                        -t ${IMAGE_NAME}:latest \
                        .
                '''
            }
        }

                stage('ECR Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-credentials',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        echo "Logging in to Amazon ECR..."

                        aws ecr get-login-password \
                            --region ${AWS_REGION} | \
                        docker login \
                            --username AWS \
                            --password-stdin ${ECR_REGISTRY}
                    '''
                }
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    echo "Pushing Seclock image to ECR..."

                    docker push ${IMAGE_NAME}:${GIT_COMMIT}
                    docker push ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {
                sh '''
                    echo "Updating Kubernetes deployment image..."

                    sed -i \
                        "s|image:.*|image: ${IMAGE_NAME}:${GIT_COMMIT}|" \
                        k8s/deployment.yaml

                    echo "Updated deployment:"
                    grep "image:" k8s/deployment.yaml
                '''
            }
        }

        stage('Commit and Push GitOps Changes') {
            steps {
                sh '''
                    git config user.name "Jenkins"
                    git config user.email "jenkins@localhost"

                    git add k8s/deployment.yaml

                    git commit \
                        -m "Update Seclock image to ${GIT_COMMIT}" \
                        || echo "No manifest changes to commit"

                    git push origin HEAD:main
                '''
            }
        }
    }

    post {
        success {
            echo '=========================================='
            echo ' SECLOCK CI/CD PIPELINE SUCCESSFUL'
            echo '=========================================='
        }

        failure {
            echo '=========================================='
            echo ' SECLOCK CI/CD PIPELINE FAILED'
            echo '=========================================='
        }

        always {
            sh '''
                docker image prune -f || true
            '''
        }
    }
}
