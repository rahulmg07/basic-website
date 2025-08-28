pipeline {
    agent any

    environment {
        IMAGE_NAME = 'website-app'
        PROD_CONTAINER_NAME = 'website-prod'
        DEV_CONTAINER_NAME = 'website-dev'
        PROD_PORT = '82'
        DEV_PORT = '8081'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checking out code from branch: \${env.BRANCH_NAME}"
                checkout scm
            }
        }

        stage('Validate') {
            steps {
                script {
                    echo "Validating repository structure..."
                    sh '''
                        if [ ! -f "index.html" ]; then
                            echo "Error: index.html not found!"
                            exit 1
                        fi

                        if grep -q "Hello world!" index.html; then
                            echo "HTML content validation passed"
                        else
                            echo "Warning: Expected content not found in index.html"
                        fi

                        echo "Repository validation completed"
                    '''
                }
            }
        }

        stage('Prepare Dockerfile') { 
            steps { 
                script { 
                    echo "Creating Dockerfile..." 
                    writeFile file: 'Dockerfile', 
                    text: ''' 
                    # Use Ubuntu as base image 
                    FROM ubuntu:20.04 
                    # Prevent interactive prompts 
                    ENV DEBIAN_FRONTEND=noninteractive 
                    # Install Apache
                    RUN apt-get update && \\ 
                        apt-get install -y apache2 && \\ 
                        apt-get clean && \\ 
                        rm -rf /var/lib/apt/lists/* 
                        
                    # Copy website files to Apache document root 
                    COPY . /var/www/html/ 
                    
                    # Set permissions 
                    RUN chown -R www-data:www-data /var/www/html && \\
                        chmod -R 755 /var/www/html 
                        
                    # Expose Apache port 
                    EXPOSE 80

                    # Start Apache in foreground 
                    CMD ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"] 
                    ''' 
                }
             }
         }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image for branch: \${env.BRANCH_NAME}"

                    def imageTag = "\${env.BRANCH_NAME}-\${env.BUILD_NUMBER}"

                    sh """
                        docker build -t \${IMAGE_NAME}:\${imageTag} .
                        docker build -t \${IMAGE_NAME}:\${env.BRANCH_NAME}-latest .

                        echo "Docker image built successfully: \${IMAGE_NAME}:\${imageTag}"
                    """

                    env.IMAGE_TAG = imageTag
                }
            }
        }

        stage('Test Build') {
            steps {
                script {
                    echo "Testing Docker image build..."

                    sh """
                        docker run -d --name test-container-\${BUILD_NUMBER} \
                            -p 8080:80 \${IMAGE_NAME}:\${env.IMAGE_TAG}

                        sleep 15

                        response=\\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080 || echo "000")

                        if [ "\\$response" = "200" ]; then
                            echo "HTTP test passed"
                        else
                            echo "HTTP test failed"
                            exit 1
                        fi

                        content=\\$(curl -s http://localhost:8080 || echo "")
                        if echo "\\$content" | grep -q "Hello world!"; then
                            echo "Content test passed"
                        else
                            echo "Content test failed"
                            exit 1
                        fi

                        docker stop test-container-\${BUILD_NUMBER}
                        docker rm test-container-\${BUILD_NUMBER}

                        echo "Build test completed successfully"
                    """
                }
            }
        }

        stage('Deploy') {
            parallel {
                stage('Deploy Development') {
                    when {
                        branch 'develop'
                    }
                    steps {
                        script {
                            echo "Deploying to Development Environment..."

                            sh """
                                docker stop \${DEV_CONTAINER_NAME} 2>/dev/null || true
                                docker rm \${DEV_CONTAINER_NAME} 2>/dev/null || true

                                docker run -d --name \${DEV_CONTAINER_NAME} \
                                    -p \${DEV_PORT}:80 \
                                    \${IMAGE_NAME}:\${env.IMAGE_TAG}

                                sleep 10
                                curl -f http://localhost:\${DEV_PORT} || exit 1

                                echo "Development deployment test completed"

                                docker stop \${DEV_CONTAINER_NAME}
                                docker rm \${DEV_CONTAINER_NAME}

                                echo "Development test container cleaned up"
                            """
                        }
                    }
                }

                stage('Deploy Production') {
                    when {
                        branch 'master'
                    }
                    steps {
                        script {
                            echo "Deploying to Production Environment..."

                            sh """
                                if docker ps -q -f name=\${PROD_CONTAINER_NAME} > /dev/null 2>&1; then
                                    echo "Backing up current production container..."
                                    backup_name="\${PROD_CONTAINER_NAME}-backup-\\$(date +%Y%m%d-%H%M%S)"
                                    docker stop \${PROD_CONTAINER_NAME}
                                    docker rename \${PROD_CONTAINER_NAME} \\$backup_name
                                fi

                                docker run -d --name \${PROD_CONTAINER_NAME} \
                                    -p \${PROD_PORT}:80 \
                                    --restart unless-stopped \
                                    --label "version=\${env.IMAGE_TAG}" \
                                    --label "deployed-by=jenkins" \
                                    \${IMAGE_NAME}:\${env.IMAGE_TAG}

                                sleep 15

                                response=\\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:\${PROD_PORT} || echo "000")

                                if [ "\\$response" = "200" ]; then
                                    echo "Production deployment successful!"
                                    echo "Website is live at http://localhost:\${PROD_PORT}"
                                else
                                    echo "Production deployment failed"
                                    exit 1
                                fi
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                if (env.BRANCH_NAME == 'master') {
                    echo "SUCCESS: Production deployment completed on port \${env.PROD_PORT}"
                } else if (env.BRANCH_NAME == 'develop') {
                    echo "SUCCESS: Development build completed - Ready for production"
                } else {
                    echo "SUCCESS: Feature branch build completed"
                }
            }
        }

        failure {
            echo "FAILURE: Pipeline failed for branch \${env.BRANCH_NAME}"
        }

        always {
            script {
                sh '''
                    docker ps -aq --filter "name=test-container" | xargs -r docker rm -f || true
                    docker image prune -f || true
                    echo "Cleanup completed"
                '''
            }
        }
    }
}