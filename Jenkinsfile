pipeline {
    agent any

    environment {
        IMAGE_NAME = 'website-app'
        PROD_CONTAINER_NAME = 'website-prod'
        PROD_PORT = '82'
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checking out branch: ${env.BRANCH_NAME}"
                checkout scm
            }
        }

        stage('Validate') {
            steps {
                script {
                    sh '''
                        if [ ! -f "index.html" ]; then
                            echo "Error: index.html not found!"
                            exit 1
                        fi
                        echo "Validation passed: index.html exists"
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                    sh """
                        docker build -t ${IMAGE_NAME}:${imageTag} .
                        docker tag ${IMAGE_NAME}:${imageTag} ${IMAGE_NAME}:${env.BRANCH_NAME}-latest
                    """
                    env.IMAGE_TAG = imageTag
                }
            }
        }

        stage('Test Build') {
            steps {
                script {
                    sh """
                        docker run -d --name test-container-${BUILD_NUMBER} -p 8080:80 ${IMAGE_NAME}:${env.IMAGE_TAG}
                        sleep 10
                        response=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080 || echo "000")
                        if [ "\$response" != "200" ]; then
                            echo "Build test failed"
                            exit 1
                        fi
                        docker stop test-container-${BUILD_NUMBER}
                        docker rm test-container-${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'master'
            }
            steps {
                script {
                    sh """
                        docker stop ${PROD_CONTAINER_NAME} 2>/dev/null || true
                        docker rm ${PROD_CONTAINER_NAME} 2>/dev/null || true
                        docker run -d --name ${PROD_CONTAINER_NAME} -p ${PROD_PORT}:80 --restart unless-stopped ${IMAGE_NAME}:${env.IMAGE_TAG}
                    """
                    echo "Production deployed successfully on port ${PROD_PORT}"
                }
            }
        }
    }

    post {
        always {
            sh '''
                docker ps -aq --filter "name=test-container" | xargs -r docker rm -f || true
                docker image prune -f || true
                echo "Cleanup completed"
            '''
        }
    }
}
