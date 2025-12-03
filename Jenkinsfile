pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                // Git 코드 가져오기 (Jenkins에서 자동으로 설정된 SCM 사용)
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh './gradlew clean test'
            }
        }
    }
}
