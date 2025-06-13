pipeline {
    agent any

    environment {
        APP_NAME = "todo-app"
        TEST_URL = "http://localhost:5000" // Pastikan ini adalah URL yang dapat diakses dari Jenkins ke aplikasi Flask Anda
        VENV_DIR = "/tmp/jenkins_venv"
        // Opsional tapi disarankan: ZAP API Key untuk keamanan.
        // Sebaiknya ini diambil dari Jenkins Credentials. Contoh: ZAP_API_KEY = credentials('ZAP_API_KEY_ID')
        ZAP_API_KEY = "" // Ganti dengan API Key ZAP Anda jika diaktifkan di ZAP. Jika tidak, biarkan kosong.
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
                    $VENV_DIR/bin/bandit -r app/ -ll -iii
                '''
            }
        }

        stage('Deploy to Test Environment') {
            steps {
                echo '🚀 Run Flask app in background'
                sh '''
                    set -e
                    # Hentikan proses Flask yang mungkin berjalan sebelumnya
                    pkill -f "flask run" || true
                    # Jalankan Flask app di background dan arahkan output ke file log
                    nohup $VENV_DIR/bin/flask run --host=0.0.0.0 > flask_app.log 2>&1 &
                    # Beri waktu aplikasi untuk memulai sepenuhnya
                    sleep 15
                '''
            }
        }

        stage('DAST Scan') {
            steps {
                echo '🛡️ Run OWASP ZAP scan'
                script {
                    sh 'docker rm -f zap || true'
                    sh """
                        docker run --name zap -u root \\
                        -v \$(pwd):/zap/wrk/:rw \\
                        -d -p 8091:8090 \\
                        ghcr.io/zaproxy/zaproxy:stable zap.sh -daemon -port 8090 -host 0.0.0.0 -config api.disablekey=false
                    """
                    echo 'Waiting for ZAP to start up and be ready...'
                    sleep 60 // Perpanjang waktu tunggu

                    // --- DEBUGGING STEP ---
                    echo 'Checking ZAP API status...'
                    sh 'curl -v http://localhost:8091/JSON/core/view/version/'
                    // Ini akan memberikan output verbose dan menunjukkan apakah koneksi berhasil atau tidak.
                    // Jika ini juga gagal, masalahnya ada pada konektivitas atau ZAP tidak mendengarkan.
                    sleep 5 // Beri waktu untuk curl selesai
                    // ----------------------

                    def zapApiKeyParam = env.ZAP_API_KEY ? "&apikey=${env.ZAP_API_KEY}" : ""

                    echo "Starting ZAP Spider for ${env.TEST_URL}..."
                    sh """
                        curl -s "http://localhost:8091/JSON/spider/action/scan/?url=${env.TEST_URL}${zapApiKeyParam}"
                    """
                    sleep 60
                    // ... (lanjutan ZAP scan dan report) ...
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo '📦 Deploy to staging (example: docker build and push)'
                sh '''
                    set -e
                    docker build -t ${APP_NAME}:latest .
                    # Anda perlu menambahkan langkah docker push ke registry di sini
                    # Contoh: docker push yourregistry/${APP_NAME}:latest
                '''
            }
        }
    }

    post {
        always {
            echo '🧹 Cleanup: stop flask app and ZAP container'
            sh 'pkill -f "flask run" || true'  // Hentikan Flask app
            sh 'docker rm -f zap || true'      // Hentikan dan hapus kontainer ZAP
            // Arsipkan laporan ZAP agar bisa diakses dari Jenkins UI
            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
        }

        failure {
            echo '❌ Build failed! Please check logs.'
        }
        success {
            echo '✅ Build successful!'
        }
    }
}