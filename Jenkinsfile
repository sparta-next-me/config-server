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

    // (수정) 매니페스트 실제 위치(레포 구조에 맞게)
    // - 지금 레포는 k8s/ 디렉토리가 없고 루트에 config-server.yaml이 있음
    MANIFEST_FILE = "config-server.yaml"
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
          ./gradlew clean test --no-daemon -Dspring.profiles.active="$TEST_PROFILE"
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
          // kubectl 설치 부분 유지
          sh '''
            set -e
            if [ ! -x "$KUBECTL_BIN" ]; then
              curl -sSL -o "$KUBECTL_BIN" "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
              chmod +x "$KUBECTL_BIN"
            fi
          '''

          withCredentials([file(credentialsId: "${KUBECONFIG_CRED_ID}", variable: 'KUBECONFIG_FILE')]) {
            sh '''
              set -e
              export KUBECONFIG="$KUBECONFIG_FILE"

              # 1) 매니페스트 적용 (Deployment/Service 생성/업데이트)
              "$KUBECTL_BIN" apply -f "$MANIFEST_FILE" -n "$NAMESPACE"

              # 2) [핵심] 이미지 업데이트 (불변 태그인 SHA를 명시적으로 주입)
              # YAML 파일에 적힌 이미지를 방금 빌드한 따끈따끈한 이미지로 교체합니다.
              "$KUBECTL_BIN" -n "$NAMESPACE" set image deployment/"$DEPLOYMENT" \
                config-server="$IMAGE_SHA"

              # 3) [무중단 확인] 새 버전이 완전히 뜰 때까지 기다립니다.
              # 우리가 YAML에 설정한 readinessProbe가 통과되어야 이 단계가 성공합니다.
              echo "Waiting for rollout to complete..."
              "$KUBECTL_BIN" -n "$NAMESPACE" rollout status deployment/"$DEPLOYMENT" --timeout=300s

              # 4) 배포 결과 로그 출력
              echo "Successfully deployed $IMAGE_SHA"
              "$KUBECTL_BIN" -n "$NAMESPACE" get pods -l app=config-server
            '''
          }
        }
      }
    }

    post {
      always {
        // 빌드 후 찌꺼기 이미지 삭제 (용량 관리)
        sh "docker rmi $IMAGE_SHA || true"
        sh "docker rmi $IMAGE_LATEST || true"
      }
    }
  }