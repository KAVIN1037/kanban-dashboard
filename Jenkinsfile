pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'daddykavin/kanban-dashboard'
        DOCKER_CREDENTIALS = 'dfd46dad-97c9-4754-860e-609e10d0bd71'

        CONTAINER_NAME = 'kanban-dashboard'
        BLUEGREEN_CONTAINER = 'kanban-dashboard-bluegreen'

        APP_PORT = '80'
        BLUEGREEN_PORT = '8084'

        PROXY_CONTAINER = 'kanban-proxy-test'
        PROXY_CONFIG = '/home/ubuntu/kanban-dashboard/nginx/default.conf'

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

        stage('Blue-Green Deploy') {
            steps {
                script {
                    env.DEPLOY_STARTED = 'true'
                }

                sh '''
                    set -e

                    IMAGE="${DOCKER_IMAGE}:build-${BUILD_NUMBER}"

                    docker pull "$IMAGE"

                    ACTIVE_CONTAINER=$(awk '/server .*:80;/{gsub("server ",""); gsub(":80;",""); print $1; exit}' "${PROXY_CONFIG}")

                    echo "Current active container: ${ACTIVE_CONTAINER}"

                    if [ "$ACTIVE_CONTAINER" = "${CONTAINER_NAME}" ]; then
                        NEW_CONTAINER="${BLUEGREEN_CONTAINER}"
                        NEW_PORT="${BLUEGREEN_PORT}"
                    else
                        NEW_CONTAINER="${CONTAINER_NAME}"
                        NEW_PORT="${APP_PORT}"
                    fi

                    echo "New container: ${NEW_CONTAINER}"
                    echo "New port: ${NEW_PORT}"

                    echo "$ACTIVE_CONTAINER" > .previous_backend
                    echo "$NEW_CONTAINER" > .new_container

                    PREVIOUS_IMAGE=$(docker inspect --format='{{.Config.Image}}' "$ACTIVE_CONTAINER" 2>/dev/null || true)
                    echo "$PREVIOUS_IMAGE" > .previous_image

                    if [ "$NEW_CONTAINER" = "$ACTIVE_CONTAINER" ]; then
                        echo "ERROR: New container is already active."
                        exit 1
                    fi

                    if docker ps -a --format '{{.Names}}' | grep -qx "$NEW_CONTAINER"; then
                        echo "Removing old inactive container: $NEW_CONTAINER"
                        docker rm -f "$NEW_CONTAINER" || true
                    fi

                    echo "Starting new container..."

                    docker run -d \
                        --name "$NEW_CONTAINER" \
                        --network kanban-bluegreen \
                        --memory 512m \
                        --cpus 0.5 \
                        -p "127.0.0.1:${NEW_PORT}:80" \
                        "$IMAGE"

                    echo "Waiting for new container to become healthy..."

                    STATUS="starting"

                    for i in $(seq 1 30); do
                        STATUS=$(docker inspect --format='{{if .State.Health}}{{.State.Health.Status}}{{else}}no-healthcheck{{end}}' "$NEW_CONTAINER" 2>/dev/null || true)

                        echo "Health status: $STATUS"

                        if [ "$STATUS" = "healthy" ]; then
                            break
                        fi

                        sleep 2
                    done

                    if [ "$STATUS" != "healthy" ]; then
                        echo "ERROR: New container did not become healthy."
                        exit 1
                    fi

                    echo "New container is healthy."

                    echo "Testing new container directly..."

                    curl -f "http://127.0.0.1:${NEW_PORT}/"

                    echo "Backing up Nginx configuration..."

                    cp "${PROXY_CONFIG}" "${PROXY_CONFIG}.rollback"

                    echo "Switching Nginx traffic to ${NEW_CONTAINER}..."

                    sed -i -E "s/server [^:;]+:80;/server ${NEW_CONTAINER}:80;/" "${PROXY_CONFIG}"

                    echo "Testing Nginx configuration..."

                    docker exec "${PROXY_CONTAINER}" nginx -t

                    echo "Reloading Nginx..."

                    docker exec "${PROXY_CONTAINER}" nginx -s reload

                    sleep 2

                    echo "Testing application through Nginx..."

                    curl -f http://127.0.0.1:8085/

                    echo "New container is serving traffic successfully."

                    echo "Stopping old container: ${ACTIVE_CONTAINER}"

                    docker stop "${ACTIVE_CONTAINER}" || true
                    docker rm "${ACTIVE_CONTAINER}" || true

                    rm -f "${PROXY_CONFIG}.rollback"

                    echo "======================================"
                    echo "ZERO/MINIMAL DOWNTIME DEPLOYMENT DONE"
                    echo "======================================"
                '''
            }
        }

        stage('Final Health Check') {
            steps {
                sh '''
                    ACTIVE_CONTAINER=$(awk '/server .*:80;/{gsub("server ",""); gsub(":80;",""); print $1; exit}' "${PROXY_CONFIG}")

                    echo "Active container: ${ACTIVE_CONTAINER}"

                    STATUS=$(docker inspect --format='{{.State.Health.Status}}' "$ACTIVE_CONTAINER")

                    echo "Container health: $STATUS"

                    if [ "$STATUS" != "healthy" ]; then
                        echo "Final health check failed."
                        exit 1
                    fi

                    curl -f http://127.0.0.1:8085/

                    echo "Final health check passed."
                '''
            }
        }
    }

    post {
        success {
            echo 'Blue-Green deployment completed successfully!'
            echo 'New container was healthy before traffic was switched.'
            echo 'Old container was stopped only after traffic verification.'
        }

        failure {
            script {
                sh '''
                    echo "Pipeline failed. Starting rollback..."

                    PREVIOUS_BACKEND=$(cat .previous_backend 2>/dev/null || true)
                    NEW_CONTAINER=$(cat .new_container 2>/dev/null || true)

                    if [ -f "${PROXY_CONFIG}.rollback" ]; then
                        echo "Restoring previous Nginx configuration..."

                        cp "${PROXY_CONFIG}.rollback" "${PROXY_CONFIG}"

                        docker exec "${PROXY_CONTAINER}" nginx -t || true
                        docker exec "${PROXY_CONTAINER}" nginx -s reload || true

                        rm -f "${PROXY_CONFIG}.rollback"
                    fi

                    if [ -n "$NEW_CONTAINER" ]; then
                        echo "Removing failed new container: $NEW_CONTAINER"
                        docker rm -f "$NEW_CONTAINER" || true
                    fi

                    if [ -n "$PREVIOUS_BACKEND" ]; then
                        if ! docker ps --format '{{.Names}}' | grep -qx "$PREVIOUS_BACKEND"; then

                            PREVIOUS_IMAGE=$(cat .previous_image 2>/dev/null || true)

                            if [ -n "$PREVIOUS_IMAGE" ]; then
                                echo "Restarting previous container: $PREVIOUS_BACKEND"

                                if [ "$PREVIOUS_BACKEND" = "${CONTAINER_NAME}" ]; then
                                    PREVIOUS_PORT="${APP_PORT}"
                                else
                                    PREVIOUS_PORT="${BLUEGREEN_PORT}"
                                fi

                                docker run -d \
                                    --name "$PREVIOUS_BACKEND" \
                                    --network kanban-bluegreen \
                                    --memory 512m \
                                    --cpus 0.5 \
                                    -p "127.0.0.1:${PREVIOUS_PORT}:80" \
                                    "$PREVIOUS_IMAGE" || true
                            fi
                        fi
                    fi

                    echo "Rollback completed."
                '''
            }
        }
    }
}
