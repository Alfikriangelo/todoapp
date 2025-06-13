pipeline {
    agent any

    environment {
        APP_NAME = "todo-app"
        // Pastikan ini adalah URL yang dapat diakses dari Jenkins ke aplikasi Flask Anda.
        // Jika Flask app dan ZAP container berada di network Docker yang sama,
        // TEST_URL mungkin perlu disesuaikan menjadi nama service Flask.
        TEST_URL = "http://localhost:5000"
        VENV_DIR = "/tmp/jenkins_venv"

        // *** PENTING: Untuk produksi, ini HARUS disimpan sebagai Jenkins Credential. ***
        // Contoh: ZAP_API_KEY = credentials('ZAP_API_KEY_ID')
        // Ganti "YOUR_ZAP_API_KEY" dengan API Key ZAP Anda yang sebenarnya jika Anda mengaktifkannya.
        // Jika Anda TIDAK ingin menggunakan API Key (tidak disarankan untuk produksi),
        // hapus '-config api.disablekey=false' dari command docker run ZAP.
        ZAP_API_KEY = "YOUR_ZAP_API_KEY" // <--- GANTI INI!
    }

    stages {
        // Stage ini tidak diperlukan jika Jenkinsfile di-checkout secara otomatis dari SCM
        // stage('Declarative: Checkout SCM') {
        //     steps {
        //         checkout scm
        //     }
        // }

        stage('Build') {
            steps {
                echo '📦 Setup virtual environment and install dependencies'
                sh '''
                    set -e
                    python3 -m venv $VENV_DIR
                    $VENV_DIR/bin/pip install --upgrade pip
                    # Perbarui Flask-SQLAlchemy ke versi yang lebih baru jika tersedia
                    # untuk mengatasi LegacyAPIWarning Query.get().
                    # Werkzeug mungkin perlu diupdate juga, atau Anda bisa coba Python 3.11 dulu.
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
                echo '🚀 Run Flask app in background for DAST scan'
                sh '''
                    set -e
                    # Hentikan proses Flask yang mungkin berjalan sebelumnya dengan lebih robust
                    pkill -f "flask run" || true
                    # Beri waktu sistem untuk membersihkan port
                    sleep 5
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
                    // Hapus kontainer ZAP sebelumnya jika ada
                    sh 'docker rm -f zap || true'

                    // Jalankan ZAP sebagai daemon. Menggunakan port 8091 di host.
                    // api.disablekey=false: Ini mengaktifkan kebutuhan API Key untuk request API ZAP.
                    sh """
                        docker run --name zap -u root \\
                        -v \$(pwd):/zap/wrk/:rw \\
                        -d -p 8091:8090 \\
                        ghcr.io/zaproxy/zaproxy:stable zap.sh -daemon -port 8090 -host 0.0.0.0 -config api.disablekey=false
                    """

                    echo 'Waiting for ZAP to start up and be ready...'
                    // Tingkatkan waktu tunggu ZAP untuk startup yang stabil
                    sleep 60 // Dari 30 detik ke 60 detik

                    // Opsional: DEBUGGING - Cek status ZAP API
                    echo 'DEBUG: Checking ZAP API version...'
                    // Ini akan gagal jika ZAP belum siap atau API Key salah
                    sh """
                        curl -s "http://localhost:8091/JSON/core/view/version/?apikey=${env.ZAP_API_KEY}"
                    """
                    sleep 5 // Beri waktu untuk debug curl selesai

                    // Tambahkan parameter API key jika diatur
                    def zapApiKeyParam = env.ZAP_API_KEY ? "&apikey=${env.ZAP_API_KEY}" : ""

                    // Lakukan spidering pada aplikasi Anda
                    echo "Starting ZAP Spider for ${env.TEST_URL}..."
                    sh """
                        curl -s "http://localhost:8091/JSON/spider/action/scan/?url=${env.TEST_URL}${zapApiKeyParam}"
                    """
                    sleep 60 // Tunggu spidering selesai. Di produksi, gunakan polling API status ZAP.

                    // Lakukan active scan
                    echo "Starting ZAP Active Scan for ${env.TEST_URL}..."
                    sh """
                        curl -s "http://localhost:8091/JSON/ascan/action/scan/?url=${env.TEST_URL}&recurse=true&inScopeOnly=true${zapApiKeyParam}"
                    """
                    sleep 120 // Tunggu active scan selesai. Di produksi, gunakan polling API status ZAP.

                    // Hasilkan laporan HTML dan simpan di direktori yang di-mount (workspace Jenkins)
                    echo 'Generating ZAP HTML report...'
                    sh """
                        curl -s "http://localhost:8091/OTHER/core/action/htmlreport/?${zapApiKeyParam}" > \${WORKSPACE}/zap_report.html
                    """
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo '📦 Deploy to staging (example: docker build and push)'
                sh '''
                    set -e
                    # Pastikan Docker Buildx terinstal untuk performa dan fitur yang lebih baik
                    # docker buildx create --use
                    docker build -t ${APP_NAME}:latest .
                    # docker push yourregistry/${APP_NAME}:latest
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
            // Perbaikan: Menggunakan allowEmptyArchive seperti yang disarankan Jenkins
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