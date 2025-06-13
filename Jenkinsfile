pipeline {
    agent any

    environment {
        APP_NAME = "todo-app"
        TEST_URL = "http://localhost:5000"
        VENV_DIR = "/tmp/jenkins_venv"
    }

    stages {

        stage('Build') {
            steps {
                echo '📦 Setup virtual environment and install dependencies'
                sh '''
                    set -e
                    python3 -m venv $VENV_DIR
                    $VENV_DIR/bin/pip install --upgrade pip
                    $VENV_DIR/bin/pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                echo '🧪 Run pytest unit tests'
                sh '''
                    set -e
                    export PYTHONPATH=.
                    $VENV_DIR/bin/pytest tests/
                '''
            }
        }

        stage('SAST Scan') {
            steps {
                echo '🔒 Run Bandit security scan'
                sh '''
                    set -e
                    $VENV_DIR/bin/bandit -r app/ -ll -iii -f json -o bandit_report.json
                    CRITICALS=$($VENV_DIR/bin/jq '.results[] | select(.issue_severity=="HIGH")' bandit_report.json | wc -l)
                    if [ "$CRITICALS" -gt 0 ]; then
                        echo "❌ Found $CRITICALS HIGH severity vulnerabilities!"
                        exit 1
                    fi
                '''
            }
        }


        stage('Deploy to Test Environment') {
            steps {
                echo '🚀 Run Flask app in background'
                sh '''
                    set -e
                    pkill -f "flask run" || true
                    $VENV_DIR/bin/flask run --host=0.0.0.0 > flask.log 2>&1 &
                    sleep 10
                '''
            }
        }

        stage('DAST Scan') {
            steps {
                echo '🛡️ Run OWASP ZAP scan'
                sh '''
                    set -e
                    docker rm -f zap || true
                    docker run --name zap -u root -v $(pwd):/zap/wrk/:rw -d -p 8091:8090 ghcr.io/zaproxy/zaproxy:stable zap.sh -daemon -port 8090 -host 0.0.0.0
                '''
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo '📦 Deploy to staging (example: docker build and push)'
                sh '''
                    set -e
                    docker build -t ${APP_NAME}:latest .
                    # docker push yourregistry/${APP_NAME}:latest
                '''
            }
        }

        stage('Post Deployment Log Review') {
            steps {
                echo '📄 Menampilkan 10 baris terakhir dari flask.log'
                sh 'tail -n 10 flask.log || true'
            }
        }

    }

    post {
        always {
            echo '🧹 Cleanup: stop flask app'
            sh 'pkill -f "flask run" || true'
            archiveArtifacts artifacts: 'logs/*.log', onlyIfSuccessful: false
        }

        failure {
            echo '❌ Build failed! Please check logs.'
        }
    }
}