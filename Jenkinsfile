pipeline {
  agent any

  stages {
    stage('Build') {
      agent { docker { image 'node:18-alpine'; reuseNode true } }
      steps { sh 'npm ci && npm run build' }
    }

    stage('Test') {
      agent { docker { image 'node:18-alpine'; reuseNode true } }
      steps { sh 'npm ci && npm test' }
      post { always { junit allowEmptyResults: true, testResults: 'test-results/**/*.xml' } }
    }

    stage('E2E (run in Docker)') {
      agent { docker { image 'mcr.microsoft.com/playwright:v1.39.0-jammy'; reuseNode true } }
      steps {
        sh '''
          set -eux
          npm ci
          npx serve -s build &
          sleep 10
          npx playwright test --reporter=html,junit
          ls -la playwright-report || true
        '''
      }
      post {
        always {
          stash name: 'e2e-html-report', includes: 'playwright-report/**', allowEmpty: true
          stash name: 'e2e-junit',       includes: '**/results.xml,**/junit*.xml', allowEmpty: true
        }
      }
    }

    stage('Deploy') {
      when { branch 'main' }
      agent { docker { image 'node:18-alpine'; reuseNode true } }
      steps { sh 'npm install -g netlify-cli && netlify --version' }
    }
  } // ← sluit 'stages' hier af!

  post {
    always {
      unstash 'e2e-html-report'
      unstash 'e2e-junit'
      script {
        if (fileExists('playwright-report/index.html')) {
          publishHTML(target: [
            allowMissing: true, alwaysLinkToLastBuild: false, keepAll: false,
            reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'E2E Report'
          ])
        } else {
          echo 'No HTML report found – skipping publishHTML'
        }
      }
      junit allowEmptyResults: true, testResults: '**/results.xml,**/junit*.xml'
      archiveArtifacts allowEmptyArchive: true, artifacts: 'playwright-report/**'
      cleanWs(deleteDirs: true, disableDeferredWipeout: true)
    }
  }
}
