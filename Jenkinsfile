pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'sidharth18/takecare:v1'
        APP_PORT = '8010'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Start Application') {
            steps {
                sh '''
                    docker compose up -d --build

                    echo "Waiting for TakeCare application..."
                    for i in $(seq 1 30); do
                        if curl -fsS http://127.0.0.1:8010/ > /dev/null; then
                            echo "TakeCare is responding."
                            exit 0
                        fi
                        sleep 5
                    done

                    echo "Application did not become ready."
                    docker compose logs
                    exit 1
                '''
            }
        }

        stage('OWASP ZAP DAST') {
            steps {
                sh '''
                    mkdir -p zap-reports
                    rm -f zap-reports/*

                    docker run --rm \
                      --network host \
                      -v "$WORKSPACE/zap-reports:/zap/wrk/:rw" \
                      zaproxy/zap-stable \
                      zap-baseline.py \
                      -t http://127.0.0.1:8010/ \
                      -m 3 -I \
                      -r zap_report_takecare.html \
                      -J zap_report_takecare.json \
                      -w zap_report_takecare.md
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        IMAGE_ID=$(docker compose images -q web | head -n 1)

                        if [ -z "$IMAGE_ID" ]; then
                            echo "TakeCare Docker image not found."
                            exit 1
                        fi

                        docker login -u "$DOCKERHUB_USER" --password-stdin <<EOFLOGIN
$DOCKERHUB_TOKEN
EOFLOGIN

                        docker tag "$IMAGE_ID" "$DOCKER_IMAGE:$BUILD_NUMBER"
                        docker tag "$IMAGE_ID" "$DOCKER_IMAGE:latest"

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
            archiveArtifacts artifacts: 'zap-reports/*',
                             allowEmptyArchive: true

            sh 'docker compose down || true'
        }

        success {
            echo 'TakeCare pipeline completed successfully!'
        }

        failure {
            echo 'TakeCare pipeline failed. Check the stage logs.'
        }
    }
}
