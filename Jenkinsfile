pipeline {
  agent any
  environment {
    DEBIAN_FRONTEND = 'noninteractive'
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install Apache') {
      steps {
        sh '''
          set -eu
          sudo apt-get update
          sudo apt-get install -y apache2 curl
          sudo systemctl enable --now apache2
          sudo systemctl status apache2 --no-pager || true
        '''
      }
    }

    stage('Smoke test') {
      steps {
        sh '''
          set -eu
          mkdir -p reports
          curl -sSf -o /tmp/index.html http://localhost/
          echo "Homepage bytes: $(wc -c < /tmp/index.html)" > reports/smoke.txt
        '''
      }
    }

    stage('Scan access log for 4xx/5xx') {
      steps {
        sh '''
          set -eu
          mkdir -p reports
          sudo tail -n 2000 /var/log/apache2/access.log > reports/access.log || true
          awk '($9 ~ /^[45][0-9][0-9]$/){c[$9]++} END{for (k in c) printf "%s %d\\n", k, c[k]}' reports/access.log \
            | sort -nr > reports/http-codes.txt || true
          f5=$(awk '$1 ~ /^5/ {s+=$2} END{print s+0}' reports/http-codes.txt)
          f4=$(awk '$1 ~ /^4/ {s+=$2} END{print s+0}' reports/http-codes.txt)
          echo "4xx=$f4 5xx=$f5" | tee reports/summary.txt
          test "${f5:-0}" -eq 0
        '''
      }
    }
  }
  post {
    always {
      archiveArtifacts artifacts: 'reports/**', fingerprint: true, onlyIfSuccessful: false
    }
  }
}
