pipeline {
    agent any

    environment {
        APP_NAME       = "config-server"
        IMAGE_NAME     = "config-server"
        IMAGE_TAG      = "latest"
        FULL_IMAGE     = "${IMAGE_NAME}:${IMAGE_TAG}"
        CONTAINER_NAME = "config-server"
        HOST_PORT      = "3100"
        CONTAINER_PORT = "3100"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

         stage('Build & Test') {
             steps {
                 sh './gradlew clean test'
                 sh './gradlew bootJar'
             }
         }

        stage('Docker Build') {
            steps {
                sh """
                  docker build -t ${FULL_IMAGE} .
                """
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
                    ${FULL_IMAGE}
                """
            }
        }
    }
}
