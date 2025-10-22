pipeline {
  agent any

  environment {
    NETLIFY_PROJECT_ID = '6d1694b2-f758-486c-8771-b6b0b74e99e1'
    NETLIFY_AUTH_TOKEN = credentials('netlify-token')
  }
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
      when { 
        expression {
          def onMain = (env.BRANCH_NAME ?: '') == 'main' || (env.GIT_BRANCH ?: '') == 'origin/main'
          def isPRtoMain = (env.CHANGE_TARGET ?: '') == 'main'
          return onMain || isPRtoMain
        }
       }
      agent { docker { image 'node:18-alpine'; reuseNode true; args '-u root' } }
      steps {
        withCredentials([string(credentialsId: 'netlify-auth-token', variable: 'NETLIFY_AUTH_TOKEN')]) {
          sh '''
            set -eux
            test -d build   # zorg dat de build-map bestaat
            npx --yes netlify-cli deploy \
              --auth "$NETLIFY_AUTH_TOKEN" \
              --site "$NETLIFY_PROJECT_ID" \
              --dir build \
              --prod \
              --message "CI ${BUILD_NUMBER}"
          '''
        }
    }
    stage('Check branch name'){
      steps {
              echo "BRANCH_NAME=${env.BRANCH_NAME}"
              echo "GIT_BRANCH=${env.GIT_BRANCH}"
              echo "CHANGE_ID=${env.CHANGE_ID}  CHANGE_TARGET=${env.CHANGE_TARGET}"
        }
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
}