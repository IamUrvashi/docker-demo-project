pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Information') {
            steps {
                wrap([$class: 'BuildUser']) {
                    script {
                        echo "=================================="
                        echo "       BUILD INFORMATION"
                        echo "=================================="

                        echo "Build triggered by: ${BUILD_USER}"
                        echo "Build User ID: ${BUILD_USER_ID}"

                        def triggeredBy = env.BUILD_USER ?: env.BUILD_USER_ID ?: 'Automated'
                        def branchName = env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'unknown'
                        branchName = branchName.replaceFirst(/^origin\\//, '')

                        currentBuild.displayName = "#${env.BUILD_NUMBER} - ${triggeredBy} - ${branchName}"
                        currentBuild.description = "Triggered by: ${triggeredBy} | Git branch: ${branchName}"

                        echo "Build Display Name: ${currentBuild.displayName}"
                        echo "Git Branch: ${env.BRANCH_NAME ?: env.GIT_BRANCH}"
                        echo "Git Commit: ${env.GIT_COMMIT}"

                        echo "=================================="
                    }
                }
            }
        }

        stage('Build Inventory Image') {
            steps {
                sh 'docker build -t inventory-service:alpine ./inventory_service'
            }
        }

        stage('Build Order Image') {
            steps {
                sh 'docker build -t order-service:alpine ./order_service'
            }
        }

        stage('Deploy Services') {
            steps {
                sh '''
                    docker network inspect coffee-shop-network >/dev/null 2>&1 || docker network create coffee-shop-network

                    docker rm -f inventory_service order_service || true

                    docker run -d \
                        --name inventory_service \
                        --restart unless-stopped \
                        --network coffee-shop-network \
                        -p 8082:8082 \
                        inventory-service:alpine

                    docker run -d \
                        --name order_service \
                        --restart unless-stopped \
                        --network coffee-shop-network \
                        -p 8083:8083 \
                        -e INVENTORY_SERVICE_URL=http://inventory_service:8082 \
                        order-service:alpine
                '''
            }
        }

        stage('Test APIs') {
            steps {
                sh '''
                    echo "Testing Inventory API..."
                    curl -f "http://host.docker.internal:8082/inventory/stock"

                    echo ""
                    echo "Testing Order API..."
                    curl -f "http://host.docker.internal:8083/order/place?coffeeType=cappuccino&quantity=1"

                    echo ""
                    echo "All API tests passed!"
                '''
            }
        }
    }
}
