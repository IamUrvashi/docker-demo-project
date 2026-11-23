pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
        AWS_REGION = 'us-west-2'
        ECR_REGISTRY = '212773812862.dkr.ecr.us-west-2.amazonaws.com'

        INVENTORY_REPO = 'inventory-service'
        ORDER_REPO = 'order-service'

        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out Coffee Shop project from GitHub...'
                checkout scm

                sh '''
                    echo "Current branch:"
                    git branch --show-current

                    echo "Latest commit:"
                    git log -1 --oneline
                '''
            }
        }

        stage('Build Information') {
            steps {
                wrap([$class: 'BuildUser']) {
                    script {
                        def branch = sh(
                            script: 'git branch --show-current',
                            returnStdout: true
                        ).trim()

                        def buildUser = env.BUILD_USER ?: 'Unknown'

                        currentBuild.displayName =
                            "#${env.BUILD_NUMBER} - ${buildUser} - ${branch}"

                        currentBuild.description =
                            "Build User: ${buildUser} | Branch: ${branch}"

                        echo "Build Number: ${env.BUILD_NUMBER}"
                        echo "Build User: ${buildUser}"
                        echo "Branch: ${branch}"
                    }
                }
            }
        }

        stage('Build Inventory Image') {
            steps {
                dir('inventory_service') {
                    sh '''
                        docker build \
                          -t ${ECR_REGISTRY}/${INVENTORY_REPO}:${IMAGE_TAG} .
                    '''
                }
            }
        }

        stage('Build Order Image') {
            steps {
                dir('order_service') {
                    sh '''
                        docker build \
                          -t ${ECR_REGISTRY}/${ORDER_REPO}:${IMAGE_TAG} .
                    '''
                }
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    aws ecr get-login-password \
                      --region ${AWS_REGION} |
                    docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh '''
                    docker push \
                      ${ECR_REGISTRY}/${INVENTORY_REPO}:${IMAGE_TAG}

                    docker push \
                      ${ECR_REGISTRY}/${ORDER_REPO}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy Using Docker Compose') {
            steps {
                sh '''
                    set -e

                    echo "Checking Docker Compose..."
                    docker compose version

                    echo "Checking ECR login..."
                    aws ecr get-login-password \
                      --region ${AWS_REGION} |
                    docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}

                    echo "Validating Compose configuration..."
                    docker compose config -q

                    echo "Removing old Coffee Shop containers..."
                    docker rm -f inventory_service order_service \
                      >/dev/null 2>&1 || true

                    echo "Pulling images from ECR..."
                    docker compose pull

                    echo "Starting services using Docker Compose..."
                    docker compose up -d --force-recreate --remove-orphans

                    echo "Checking running services..."
                    docker compose ps --status running -q inventory-service | grep -q .
                    docker compose ps --status running -q order-service | grep -q .

                    docker compose ps

                    echo "Deployment successful!"
                '''
            }
        }

        stage('Test APIs') {
            steps {
                sh '''
                    echo "Waiting for Inventory API..."

                    for i in $(seq 1 10); do
                        if curl -fsS "http://localhost:8082/inventory/stock"; then
                            echo "Inventory API is working!"
                            break
                        fi

                        if [ "$i" -eq 10 ]; then
                            echo "Inventory API failed!"
                            exit 1
                        fi

                        sleep 3
                    done

                    echo ""
                    echo "Testing Order API..."

                    for i in $(seq 1 10); do
                        if curl -fsS "http://localhost:8083/order/place?coffeeType=cappuccino&quantity=1"; then
                            echo "Order API is working!"
                            break
                        fi

                        if [ "$i" -eq 10 ]; then
                            echo "Order API failed!"
                            exit 1
                        fi

                        sleep 3
                    done

                    echo ""
                    echo "Both APIs are working successfully!"
                '''
            }
        }
    }

    post {
        success {
            echo 'Coffee Shop ECR + Docker Compose CI/CD Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check Console Output.'
        }

        always {
            echo "Pipeline execution completed. Build: ${env.BUILD_NUMBER}"
        }
    }
}
