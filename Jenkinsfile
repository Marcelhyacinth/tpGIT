pipeline {
    agent any

    environment {
        HARBOR_URL = "localhost:8081"
        PROJECT    = "projet-ci-cd"
        IMAGE_NAME = "fastapi-app"
        IMAGE_TAG  = "${BUILD_NUMBER}"
        FULL_IMAGE = "${HARBOR_URL}/${PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SAST - Bandit') {
            steps {
                sh 'bandit -r src/ -f txt -o bandit-report.txt || true'
                sh 'cat bandit-report.txt'
            }
        }

        stage('Secrets Scan - Gitleaks') {
            steps {
                sh 'gitleaks detect --source . --report-path gitleaks-report.json || true'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${FULL_IMAGE} -f docker/Dockerfile .'
            }
        }

        stage('Scan Vulnérabilités - Trivy') {
            steps {
                sh 'trivy image --exit-code 0 --severity HIGH,CRITICAL ${FULL_IMAGE}'
            }
        }

        stage('Push vers Harbor') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'harbor-credentials',
                    usernameVariable: 'HARBOR_USER',
                    passwordVariable: 'HARBOR_PASS'
                )]) {
                    sh '''
                        docker login ${HARBOR_URL} -u ${HARBOR_USER} -p ${HARBOR_PASS}
                        docker push ${FULL_IMAGE}
                    '''
                }
            }
        }

        stage('Signature - Cosign') {
            steps {
                sh 'cosign sign --insecure-skip-verify ${FULL_IMAGE} || true'
            }
        }

        stage('Déploiement Docker Compose') {
            steps {
                sh '''
                    export IMAGE=${FULL_IMAGE}
                    docker compose -f docker-compose.yml up -d --force-recreate
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '*-report.*', allowEmptyArchive: true
        }
        success {
            echo '✅ Pipeline réussi !'
        }
        failure {
            echo '❌ Pipeline échoué !'
        }
    }
}
