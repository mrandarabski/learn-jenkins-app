pipeline {
  agent any
  options { skipDefaultCheckout(true) }   // ➋: wij doen zelf cleanup + checkout

  environment {
    NETLIFY_PROJECT_ID = '6d1694b2-f758-486c-8771-b6b0b74e99e1'
    // Haal het token NIET hier én via withCredentials; kies één plek.
    // We gebruiken straks alleen withCredentials in Deploy.
  }

  stages {
    stage('Checkout (clean & fix perms)') {
      steps {
        // ➋: verwijder rommel van vorige run (kan falen bij root-owned files)
        deleteDir()
        // herstel permissies / verwijder hardnekkige .netlify resten
        sh '''
          set -eux
          (chmod -R u+rwX . || true)
          rm -rf .netlify || true
        '''
        checkout scm
      }
    }

    stage('Build') {
      agent { docker { image 'node:18-alpine'; reuseNode true; args '-u 1000:1000' } } // ➊ niet als root
      steps { sh 'npm ci && npm run build' }
    }

    stage('Test') {
      agent { docker { image 'node:18-alpine'; reuseNode true; args '-u 1000:1000' } } // ➊
      steps { sh 'npm ci && npm test' }
      post { always { junit allowEmptyResults: true, testResults: 'test-results/**/*.xml' } }
    }

    stage('E2E (run in Docker)') {
      agent { docker { image 'mcr.microsoft.com/playwright:v1.39.0-jammy'; reuseNode true; args '-u 1000:1000' } } // ➊
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
      when { branch 'main' }  // houd je eigen gating aan
      agent { docker { image 'node:18'; reuseNode true; args '-u 1000:1000' } } // ➊ Debian-based met bash
      steps {
        // ➌ gebruik één credentials ID en gebruik met withCredentials
        withCredentials([string(credentialsId: 'netlify-token', variable: 'NETLIFY_AUTH_TOKEN')]) {
          sh '''
            set -eux
            test -d build  # build/ moet bestaan uit eerdere stage
            npx --yes netlify-cli deploy \
              --auth "$NETLIFY_AUTH_TOKEN" \
              --site "$NETLIFY_PROJECT_ID" \
              --dir build \
              --prod \
              --message "CI ${BUILD_NUMBER}"
          '''
        }
      }
    }

    stage('Check branch name') {
      steps {
        echo "BRANCH_NAME=${env.BRANCH_NAME}"
        echo "GIT_BRANCH=${env.GIT_BRANCH}"
        echo "CHANGE_ID=${env.CHANGE_ID}  CHANGE_TARGET=${env.CHANGE_TARGET}"
      }
    }
  } // einde stages

  post {
    always {
      // ➍ zorg voor workspace-context; unstash defensief
      node(env.NODE_NAME ?: null) {
        script {
          try { unstash 'e2e-html-report' } catch (ignore) { echo 'No e2e-html-report stash' }
          try { unstash 'e2e-junit' }       catch (ignore) { echo 'No e2e-junit stash' }

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
