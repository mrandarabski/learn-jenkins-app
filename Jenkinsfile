pipeline {
  agent any
   stages {
        stage('Test HTML plugin') {
            steps {
                publishHTML(target: [
                    reportDir: 'playwright-report',
                    reportFiles: 'index.html',
                    reportName: 'HTML Report'
                ])
            }
        }
    }  
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

    stage('E2E') {
      agent { docker { image 'mcr.microsoft.com/playwright:v1.39.0-jammy'; reuseNode true } }
      steps {
        sh '''
          npm ci
          npx serve -s build &
          sleep 10
          npx playwright test --reporter=html,junit || true
        '''
      }
      post {
        always {
          publishHTML(
            allowMissing: true,
            alwaysLinkToLastBuild: false,
            keepAll: false,
            reportDir: 'playwright-report',
            reportFiles: 'index.html',
            reportName: 'E2E Report'
          )
          junit allowEmptyResults: true, testResults: '**/results.xml'
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
  }

  post {
    always {
      // bv. notificaties/cleanup
       cleanWs()
       // het eind
    }
  }
}
