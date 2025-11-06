pipeline {
  agent { label 'docker-agent' } // ensure agent has docker + java/maven OR adjust agent

  environment {
    APP_PORT = 8080
    APP_URL  = "http://127.0.0.1:${APP_PORT}"
  }

  options { timeout(time: 20, unit: 'MINUTES'); ansiColor('xterm') }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Build') {
      steps {
        sh 'mvn -B -DskipTests clean package'
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      }
    }

    stage('SAST - Semgrep (fast)') {
      steps {
        sh '''
          if ! command -v semgrep >/dev/null 2>&1; then
            pip3 install --user semgrep
            export PATH=$HOME/.local/bin:$PATH
          fi
          semgrep --config=auto --json > semgrep-results.json || true
        '''
        archiveArtifacts artifacts: 'semgrep-results.json', allowEmptyArchive: true
      }
    }

    stage('Start App') {
      steps {
        sh '''
          nohup java -jar target/*.jar > app.log 2>&1 & echo $! > app.pid
          for i in $(seq 1 30); do
            if curl -sSf ${APP_URL} >/dev/null 2>&1; then break; fi
            sleep 2
          done
          tail -n 50 app.log || true
        '''
      }
    }

    stage('DAST - OWASP ZAP (baseline)') {
      steps {
        sh '''
          docker run --rm --network host -v $PWD:/zap/wrk/:rw owasp/zap2docker-stable \
            zap-baseline.py -t ${APP_URL} -r zap-report.html -J zap-report.json || true
        '''
        archiveArtifacts artifacts: 'zap-report.html, zap-report.json', allowEmptyArchive: true
      }
    }
  }

  post {
    always {
      sh 'if [ -f app.pid ]; then kill $(cat app.pid) || true; rm -f app.pid; fi' 
      archiveArtifacts artifacts: 'app.log', allowEmptyArchive: true
      cleanWs()
    }
  }
}
