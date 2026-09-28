pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'daddykavin/kanban-dashboard'
        DOCKER_CREDENTIALS = 'dfd46dad-97c9-4754-860e-609e10d0bd71'
        CONTAINER_NAME = 'kanban-dashboard'
        APP_PORT = '80'
        DEPLOY_STARTED = 'false'
        GIT_SHA = ''
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm
          script {
            env.GIT_SHA = sh(
                script: 'git rev-parse --short HEAD',
                returnStdout: true
            ).trim()

            echo "Git SHA: ${env.GIT_SHA}"
        }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
            docker build -t ${DOCKER_IMAGE}:build-${BUILD_NUMBER} .
            docker tag ${DOCKER_IMAGE}:build-${BUILD_NUMBER} ${DOCKER_IMAGE}:${GIT_SHA}
            '''
          }
        }

        stage('Registry Login and Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                        docker push ${DOCKER_IMAGE}:build-${BUILD_NUMBER}
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    env.DEPLOY_STARTED = 'true'
                }

                sh '''
                    PREVIOUS_IMAGE=$(docker inspect --format='{{.Config.Image}}' ${CONTAINER_NAME} 2>/dev/null || true)
                    echo "$PREVIOUS_IMAGE" > .previous_image

                    docker pull ${DOCKER_IMAGE}:build-${BUILD_NUMBER}

                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true

                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --memory 512m \
                        --cpus 0.5 \
                        -p ${APP_PORT}:80 \
                        ${DOCKER_IMAGE}:build-${BUILD_NUMBER}
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    sleep 10

                    STATUS=$(docker inspect --format='{{.State.Health.Status}}' ${CONTAINER_NAME})

                    echo "Container health: $STATUS"

                    if [ "$STATUS" != "healthy" ]; then
                        echo "Health check failed"
                        exit 1
                    fi

                    curl -f http://localhost:${APP_PORT}
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            script {
                def previousImage = sh(
                    script: "test -s .previous_image && cat .previous_image || true",
                    returnStdout: true
                ).trim()

                if (env.DEPLOY_STARTED == 'true' && previousImage) {
                    echo "Rolling back to: ${previousImage}"

                    withEnv(["ROLLBACK_IMAGE=${previousImage}"]) {
                        sh '''
                            docker stop ${CONTAINER_NAME} || true
                            docker rm ${CONTAINER_NAME} || true

                            docker run -d \
                                 --name ${CONTAINER_NAME} \
                                 --memory 512m \
                                 --cpus 0.5 \
                                 -p ${APP_PORT}:80 \
                                  $ROLLBACK_IMAGE
                        '''
                    }
                } else {
                    echo 'No previous image available for rollback.'
                }

                echo 'Pipeline failed. Rollback attempted.'
            }
        }
    }
}
