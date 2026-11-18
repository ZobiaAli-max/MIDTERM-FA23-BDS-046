pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 5, unit: 'MINUTES')
    }

    environment {
        IMAGE_NAME = 'midterm-fa23-bds-046'
        CONTAINER_NAME = 'midterm-fa23-bds-046'
    }

    stages {
        stage('Checkout Latest Code') {
            steps {
                checkout scm
            }
        }

        stage('Train Model') {
            steps {
                sh '''
                    set -e
                    python3 -m venv .venv
                    .venv/bin/python -m pip install -r requirements.txt
                    GIT_COMMIT=$(git rev-parse HEAD) .venv/bin/python train.py
                '''
            }
        }

        stage('Test Model and API') {
            steps {
                sh '''
                    set -e
                    .venv/bin/python -m compileall -q app.py train.py
                    .venv/bin/python - <<'PY'
import json
import joblib
from pathlib import Path

m = json.loads(Path("artifacts/metrics.json").read_text())
bundle = joblib.load("artifacts/model.joblib")

assert m["student_id"] == "FA23-BDS-046"
assert m["training_rows"] >= 100
assert 0 <= m["accuracy"] <= 1
assert m["git_sha"] == __import__("subprocess").check_output(
    ["git", "rev-parse", "HEAD"], text=True
).strip()
assert len(bundle["features"]) == 4
print("Model validation passed")
PY
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Deploy FastAPI') {
            steps {
                sh '''
                    set -e
                    docker rm -f ${CONTAINER_NAME} || true
                    docker run -d \
                      --name ${CONTAINER_NAME} \
                      --restart unless-stopped \
                      -p 8000:8000 \
                      ${IMAGE_NAME}:${BUILD_NUMBER}

                    sleep 5
                    curl --fail http://localhost:8000/metrics
                '''
            }
        }
    }

    post {
    always {
        emailext(
            to: 'qasimalik@gmail.com',
            from: 'zobiaakbarali@gmail.com',
            subject: "FA23-BDS-046 - Jenkins Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}",
            body: """Student ID: FA23-BDS-046
Build Number: ${env.BUILD_NUMBER}
Build Status: ${currentBuild.currentResult}
Git Commit: ${env.GIT_COMMIT}
Build URL: ${env.BUILD_URL}

Automated Jenkins CI/CD Pipeline Notification."""
        )
    }
}
