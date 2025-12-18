pipeline {
    agent any

    environment {
        APP_NAME        = "config-server"

        // GHCR에 올릴 이미지 이름
        REGISTRY        = "ghcr.io"
        GH_OWNER        = "sparta-next-me"
        IMAGE_REPO      = "config-server"
        FULL_IMAGE      = "${REGISTRY}/${GH_OWNER}/${IMAGE_REPO}:latest"

        CONTAINER_NAME  = "config-server"
        HOST_PORT       = "3100"
        CONTAINER_PORT  = "3100"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh '''
                  ./gradlew clean test --no-daemon
                  ./gradlew bootJar --no-daemon
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                  docker build -t ${FULL_IMAGE} .
                """
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'ghcr-credential',
                        usernameVariable: 'REGISTRY_USER',
                        passwordVariable: 'REGISTRY_TOKEN'
                    )
                ]) {
                    sh """
                      echo "$REGISTRY_TOKEN" | docker login ghcr.io -u "$REGISTRY_USER" --password-stdin
                      docker push ${FULL_IMAGE}
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                sh """
                  # 기존 컨테이너 있으면 정지/삭제
                  if [ \$(docker ps -aq -f name=${CONTAINER_NAME}) ]; then
                    echo "Stopping existing container..."
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                  fi

                  echo "Starting new container..."
                  docker run -d --name ${CONTAINER_NAME} \\
                    -p ${HOST_PORT}:${CONTAINER_PORT} \\
                    -e SPRING_PROFILES_ACTIVE=prod \\
                    ${FULL_IMAGE}
                """
            }
        }
    }
}
