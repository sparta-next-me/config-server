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

    // kubeconfig Jenkins Credentials(file) ID
    KUBECONFIG_CRED_ID = "k3s-kubeconfig"

    // (권장) CI 테스트는 외부 의존 끊기 위해 test 프로파일 강제
    TEST_PROFILE = "test"

    // kubectl을 /usr/local/bin 대신 워크스페이스에 설치 (권한 이슈 방지)
    KUBECTL_BIN = "${WORKSPACE}/kubectl"
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
        // 테스트는 무조건 test 프로파일로 실행(외부: Eureka/Loki/Config Git clone 등 차단)
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

          // 실제 푸시/배포에 사용할 이미지 풀네임
          env.IMAGE_SHA    = "${REGISTRY}/${GH_OWNER}/${IMAGE_REPO}:${env.IMAGE_TAG}"
          env.IMAGE_LATEST = "${REGISTRY}/${GH_OWNER}/${IMAGE_REPO}:latest"
        }

        echo "IMAGE_SHA=${env.IMAGE_SHA}"
        echo "IMAGE_LATEST=${env.IMAGE_LATEST}"
      }
    }

    stage('Docker Build') {
      steps {
        // 이미지 빌드: SHA 태그로 생성 + 편의용 latest 태그도 함께 지정
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
          // GHCR 로그인 후 SHA/Latest 둘 다 푸시
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
        // kubectl 설치 (권한 문제 피하려고 WORKSPACE에 설치)
        sh '''
          set -e
          if [ ! -x "$KUBECTL_BIN" ]; then
            echo "kubectl not found in workspace. installing..."
            curl -sSL -o "$KUBECTL_BIN" "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
            chmod +x "$KUBECTL_BIN"
          fi
        '''

        // kubeconfig 파일을 Jenkins Credentials에서 꺼내 kubectl이 사용하도록 설정
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED_ID}", variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            export KUBECONFIG="$KUBECONFIG_FILE"

            # 1) 매니페스트 적용 (Deployment/Service 존재 보장)
            #   - k8s/config-server.yaml 안에 namespace가 이미 있으면 -n은 생략하는 게 깔끔
            "$KUBECTL_BIN" apply -f k8s/config-server.yaml

            # 2) 실제 배포 이미지를 SHA 태그로 고정 업데이트 (최신 반영 가장 확실)
            "$KUBECTL_BIN" -n "$NAMESPACE" set image deployment/"$DEPLOYMENT" \
              config-server="$IMAGE_SHA"

            # 3) 롤아웃 완료까지 대기
            "$KUBECTL_BIN" -n "$NAMESPACE" rollout status deployment/"$DEPLOYMENT" --timeout=180s

            # 4) 결과 확인
            "$KUBECTL_BIN" -n "$NAMESPACE" get pods -l app=config-server -o wide
            "$KUBECTL_BIN" -n "$NAMESPACE" get deploy "$DEPLOYMENT" -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
          '''
        }
      }
    }
  }
}
