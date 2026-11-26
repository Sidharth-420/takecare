
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
        DOCKER_IMAGE = 'sidharth18/takecare'
        APP_PORT = '8010'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SAST - Semgrep') {
            steps {
                sh '''
                    mkdir -p security-reports

                    docker run --rm \
                      -v "$WORKSPACE:/src" \
                      -w /src \
                      semgrep/semgrep \
                      semgrep scan \
                      --config auto \
                      --json \
                      --output security-reports/semgrep-report.json .
                '''
            }
        }

        stage('Dependency Scan - Trivy') {
            steps {
                sh '''
                    docker run --rm \
                      -v "$WORKSPACE:/src" \
                      aquasec/trivy:latest \
                      fs \
                      --scanners vuln \
                      --format json \
                      --output /src/security-reports/trivy-report.json \
                      /src
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t "$DOCKER_IMAGE:$BUILD_NUMBER" \
                      .
                '''
            }
        }

        stage('Start Application') {
            steps {
                sh '''
                    docker compose up -d --build

                    echo "Waiting for TakeCare..."

                    for i in $(seq 1 30); do
                        if curl -fsS http://127.0.0.1:$APP_PORT/ > /dev/null; then
                            echo "TakeCare is responding."
                            exit 0
                        fi
                        sleep 5
                    done

                    docker compose logs
                    exit 1
                '''
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                sh '''
                    mkdir -p zap-reports

                    docker run --rm \
                      --network host \
                      -v "$WORKSPACE/zap-reports:/zap/wrk/:rw" \
                      zaproxy/zap-stable \
                      zap-baseline.py \
                      -t http://127.0.0.1:$APP_PORT/ \
                      -m 3 -I \
                      -r zap-report.html \
                      -J zap-report.json \
                      -w zap-report.md
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        set +x

                        echo "$DOCKERHUB_TOKEN" | docker login \
                          -u "$DOCKERHUB_USER" \
                          --password-stdin

                        docker tag "$DOCKER_IMAGE:$BUILD_NUMBER" \
                          "$DOCKER_IMAGE:latest"

                        docker push "$DOCKER_IMAGE:$BUILD_NUMBER"
                        docker push "$DOCKER_IMAGE:latest"

                        docker logout
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'security-reports/*,zap-reports/*',
                             allowEmptyArchive: true

            sh 'docker compose down || true'
        }

        success {
            echo 'TakeCare CI/CD pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check the stage logs and security reports.'
        }
    }
}
