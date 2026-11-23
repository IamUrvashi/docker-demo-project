
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
        NETWORK_NAME = 'coffee-shop-network'

        INVENTORY_IMAGE = 'inventory-service:alpine'
        ORDER_IMAGE = 'order-service:alpine'

        INVENTORY_CONTAINER = 'inventory_service'
        ORDER_CONTAINER = 'order_service'

        INVENTORY_PORT = '8082'
        ORDER_PORT = '8083'
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
                echo 'Building Inventory Service Docker image...'

                dir('inventory_service') {
                    sh '''
                        docker build -t ${INVENTORY_IMAGE} .
                    '''
                }
            }
        }

        stage('Build Order Image') {
            steps {
                echo 'Building Order Service Docker image...'

                dir('order_service') {
                    sh '''
                        docker build -t ${ORDER_IMAGE} .
                    '''
                }
            }
        }

        stage('Deploy Services') {
            steps {
                echo 'Checking Docker network...'

                sh '''
                    docker network inspect ${NETWORK_NAME} \
                    >/dev/null 2>&1 || docker network create ${NETWORK_NAME}
                '''

                echo 'Removing old containers...'

                sh '''
                    docker rm -f ${INVENTORY_CONTAINER} || true
                    docker rm -f ${ORDER_CONTAINER} || true
                '''

                echo 'Deploying Inventory Service...'

                sh '''
                    docker run -d \
                        --name ${INVENTORY_CONTAINER} \
                        --restart unless-stopped \
                        --network ${NETWORK_NAME} \
                        -p ${INVENTORY_PORT}:8082 \
                        ${INVENTORY_IMAGE}
                '''

                echo 'Deploying Order Service...'

                sh '''
                    docker run -d \
                        --name ${ORDER_CONTAINER} \
                        --restart unless-stopped \
                        --network ${NETWORK_NAME} \
                        -p ${ORDER_PORT}:8083 \
                        -e INVENTORY_SERVICE_URL=http://inventory_service:8082 \
                        ${ORDER_IMAGE}
                '''

                echo 'Deployment completed.'

                sh 'docker ps'
            }
        }

        stage('Test APIs') {
            steps {
                echo 'Testing Inventory and Order APIs...'

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
            echo 'Coffee Shop CI/CD Pipeline completed successfully!'
        }

        failure {
            echo 'Coffee Shop CI/CD Pipeline failed. Check Console Output.'
        }

        always {
            echo "Pipeline execution completed. Build: ${env.BUILD_NUMBER}"
        }
    }
}

