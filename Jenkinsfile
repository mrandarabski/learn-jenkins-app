pipeline {
  agent any 
  stages {
    stage('Build') {
      agent { docker { image 'node:18-alpine'; reuseNode true } }
      steps {
        sh '''
          npm ci
          npm run build
        '''
      }
    }
    stage('Test') {
      agent { docker { image 'node:18-alpine'; reuseNode true } }
      steps {
        sh '''
          npm ci
          npm test
        '''
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'test-results/**/*.xml'
        }
      }
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
          # verifiëren en tonen wat er is gemaakt
          pwd; ls -la
          test -d playwright-report
          ls -la playwright-report
        '''
      }

      post {
        always {
          // maak rapporten beschikbaar buiten Docker
          stash name: 'e2e-html-report', includes: 'playwright-report/**', allowEmpty: true
          stash name: 'e2e-junit',       includes: '**/results.xml',     allowEmpty: true
        }
      }
    }

    stage('Deploy') {
      when { branch 'main' } // optioneel
      agent { docker { image 'node:18-alpine'; reuseNode true } }
      steps {
        sh '''
          npm install -g netlify-cli
          netlify --version
          # netlify deploy --auth $NETLIFY_TOKEN --site $NETLIFY_SITE_ID --dir build --prod
        '''
      }
    }
     // Nieuwe stage: draait NIET in Docker
    post {
      always {
        unstash 'e2e-html-report'
        unstash 'e2e-junit'

        // publiceer HTML-rapport veilig buiten Docker
        script {
          if (fileExists('playwright-report/index.html')) {
            publishHTML(target: [
              allowMissing: true,
              alwaysLinkToLastBuild: false,
              keepAll: false,
              reportDir: 'playwright-report',
              reportFiles: 'index.html',
              reportName: 'E2E Report'
            ])
          } else {
            echo 'No HTML report found – skipping publishHTML'
          }
        }

         junit allowEmptyResults: true, testResults: '**/results.xml'
         archiveArtifacts allowEmptyArchive: true, artifacts: 'playwright-report/**'

         cleanWs(deleteDirs: true, disableDeferredWipeout: true)
      }
    }
  }
}
