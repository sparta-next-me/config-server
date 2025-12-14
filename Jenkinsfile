pipeline {
  agent any

  environment {
    // ===== GHCR 레지스트리 설정 =====
    REGISTRY   = "ghcr.io"
    GH_OWNER   = "sparta-next-me"
    IMAGE_REPO = "config-server"

    // ===== K8s 배포 타겟 =====
    NAMESPACE  = "next-me"
    DEPLOYMENT = "config-server"

    // Jenkins Credentials에 등록해둔 kubeconfig 파일(ID)
    // (eureka에서 쓰던 방식과 동일)
    KUBECONFIG_CRED_ID = "k3s-kubeconfig"
  }

  stages {

    stage('Checkout') {
      steps {
        // Git 저장소 체크아웃
        checkout scm
      }
    }

    stage('Build & Test') {
      steps {
        // Gradle 테스트 + Jar 빌드
        sh '''
          set -e
          ./gradlew clean test --no-daemon
          ./gradlew bootJar --no-daemon
        '''
      }
    }

    stage('Compute Image Tags') {
      steps {
        script {
          // 커밋 SHA를 태그로 사용 (불변 태그)
          def sha = sh(script: "git rev-parse --short=12 HEAD", returnStdout: true).trim()
          env.IMAGE_TAG = sha

          // 실제로 푸시할 이미지 풀네임들
          env.IMAGE_SHA   = "${REGISTRY}/${GH_OWNER}/${IMAGE_REPO}:${env.IMAGE_TAG}"
          env.IMAGE_LATEST= "${REGISTRY}/${GH_OWNER}/${IMAGE_REPO}:latest"
        }

        // 어떤 태그로 빌드/배포하는지 로그로 남김
        echo "IMAGE_SHA=${env.IMAGE_SHA}"
        echo "IMAGE_LATEST=${env.IMAGE_LATEST}"
      }
    }

    stage('Docker Build') {
      steps {
        // 이미지 빌드: SHA 태그로 생성 + 최신 편의용으로 latest도 같이 태그
        sh '''
          set -e
          docker build -t "$IMAGE_SHA" .
          docker tag "$IMAGE_SHA" "$IMAGE_LATEST"
        '''
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
          // GHCR 로그인 후 두 태그(SHA, latest) 모두 푸시
          sh '''
            set -e
            echo "$REGISTRY_TOKEN" | docker login ghcr.io -u "$REGISTRY_USER" --password-stdin
            docker push "$IMAGE_SHA"
            docker push "$IMAGE_LATEST"
          '''
        }
      }
    }

    stage('Deploy to k3s') {
      steps {
        // kubectl이 없으면 설치(환경에 따라 권한 문제 있을 수 있음)
        sh '''
          set -e
          if ! command -v kubectl >/dev/null 2>&1; then
            echo "kubectl not found. installing..."
            curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
            chmod +x kubectl
            mv kubectl /usr/local/bin/kubectl
          fi
        '''

        // kubeconfig 파일을 Jenkins Credentials에서 꺼내서 kubectl이 사용하게 함
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED_ID}", variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            export KUBECONFIG="$KUBECONFIG_FILE"

            # 1) 매니페스트 적용(Deployment/Service 등 존재 보장)
            kubectl apply -f k8s/config-server.yaml -n "$NAMESPACE"

            # 2) 배포 이미지 "문자열 자체"를 SHA 태그로 업데이트 (최신 반영을 가장 확실하게 보장)
            kubectl -n "$NAMESPACE" set image deployment/"$DEPLOYMENT" \
              config-server="$IMAGE_SHA" --record

            # 3) 롤아웃 완료까지 대기 (실패하면 파이프라인도 실패하도록 둠)
            kubectl -n "$NAMESPACE" rollout status deployment/"$DEPLOYMENT" --timeout=180s

            # 4) 결과 확인 로그
            kubectl -n "$NAMESPACE" get pods -l app=config-server -o wide
            kubectl -n "$NAMESPACE" describe deploy/"$DEPLOYMENT" | egrep "Image:|Image Pull Policy:" || true
          '''
        }
      }
    }
  }
}
