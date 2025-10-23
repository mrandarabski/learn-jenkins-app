// ===== Declarative Jenkins Pipeline =====
// Deze pipeline bouwt, test, voert E2E-tests uit, publiceert rapporten en
// deployt naar Netlify (alleen op de main branch).

pipeline {
  agent any // gebruik elke beschikbare Jenkins-agent (node)

  // ---------- Omgevingsvariabelen ----------
  environment {
    NETLIFY_LOG = 'debug'
    // ID van je Netlify-site (vind je in je Netlify dashboard)
    NETLIFY_PROJECT_ID = '6d1694b2-f758-486c-8771-b6b0b74e99e1'
    // Jenkins-credential die je in Credentials hebt aangemaakt
    NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    JENKINS_UID = '1000'
    JENKINS_GID = '1000'
  }

  stages {

    // ---------- STAGE 1: Build ----------
    stage('Build') {
      agent {
        docker {
          image 'node:18-alpine'     // lichtgewicht Node-image
          reuseNode true             // hergebruik dezelfde Jenkins-node
          args "-u ${JENKINS_UID}:${JENKINS_GID}" // voorkom root-owned bestanden
          // het werkt nog niet
        }
      }
      steps {
        sh '''
          set -eux
          npm ci                    # schone installatie van dependencies
          npm run build              # bouw productieversie (bijv. React)
        '''
      }
    }

    // ---------- STAGE 2: Unit Tests ----------
    stage('Test') {
      agent {
        docker {
          image 'node:18-alpine'
          reuseNode true
          args "-u ${JENKINS_UID}:${JENKINS_GID}"
        }
      }
      steps {
        sh '''
          set -eux
          npm ci
          npm test || true           # laat testfouten niet meteen pipeline breken
        '''
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'test-results/**/*.xml'
        }
      }
    }

    // ---------- STAGE 3: End-to-End Tests ----------
    stage('E2E (Playwright)') {
      agent {
        docker {
          image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
          reuseNode true
          args "-u ${JENKINS_UID}:${JENKINS_GID}"
        }
      }
      steps {
        sh '''
          set -eux
          npm ci
          npx serve -s build &       # start lokale webserver voor tests
          sleep 10
          npx playwright test --reporter=html,junit
          ls -la playwright-report || true
        '''
      }
      post {
        always {
          // bewaar de rapporten zodat andere stages ze kunnen gebruiken
          stash name: 'e2e-html-report', includes: 'playwright-report/**', allowEmpty: true
          stash name: 'e2e-junit', includes: '**/results.xml,**/junit*.xml', allowEmpty: true
        }
      }
    }

    // ---------- STAGE 4: Rapporten publiceren ----------
    stage('Publish Reports') {
      steps {
        // haal rapporten terug uit stashes
        unstash 'e2e-html-report'
        unstash 'e2e-junit'

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
            echo '⚠️ Geen HTML rapport gevonden - overslaan publishHTML'
          }
        }

        // combineer testresultaten
        junit allowEmptyResults: true, testResults: '**/results.xml,**/junit*.xml'
        // archiveer HTML rapport
        archiveArtifacts allowEmptyArchive: true, artifacts: 'playwright-report/**'
      }
    }
    // ---------- STAGE 5: Netlify diagnostics ----------
    stage('Netlify diagnostics') {
      steps {
        withCredentials([string(credentialsId: 'netlify-token', variable: 'NETLIFY_AUTH_TOKEN')]) {
          sh '''
            set -eux

            echo "Netlify CLI version:"
            npx --yes netlify --version

            echo "Token present? (masked length):"
            [ -n "$NETLIFY_AUTH_TOKEN" ] && echo "NETLIFY_AUTH_TOKEN set (len=${#NETLIFY_AUTH_TOKEN})" || echo "MISSING TOKEN"

            echo "List sites available to this token (first 10 lines):"
            npx --yes netlify sites:list --json --auth "$NETLIFY_AUTH_TOKEN" | head -n 10 || true

            echo "Check that our site id is visible in the list:"
            npx --yes netlify sites:list --json --auth "$NETLIFY_AUTH_TOKEN" | grep -n "\"id\": \"${NETLIFY_PROJECT_ID}\"" || true

            echo "Environment variables for the site (requires token + site):"
            npx --yes netlify env:list --auth "$NETLIFY_AUTH_TOKEN" --site "$NETLIFY_PROJECT_ID" || true
          '''
        }
      }
    }

    // ---------- STAGE 6: Deploy ----------
    stage('Deploy') {
      when {
        branch 'main' // alleen deployen op main branch
      }
      agent {
        docker {
          image 'node:18' // Debian-based image (bevat bash)
          reuseNode true
          args "-u ${JENKINS_UID}:${JENKINS_GID}"
        }
      }
      steps {
        withCredentials([string(credentialsId: 'netlify-token', variable: 'NETLIFY_AUTH_TOKEN')]) {
          sh '''
            set -eux
            test -d build || { echo "❌ Build-map ontbreekt"; exit 1; }
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

    // ---------- STAGE 7: Branch Info ----------
    stage('Check Branch Info') {
      steps {
        echo "BRANCH_NAME=${env.BRANCH_NAME}"
        echo "GIT_BRANCH=${env.GIT_BRANCH}"
        echo "CHANGE_ID=${env.CHANGE_ID}  CHANGE_TARGET=${env.CHANGE_TARGET}"
      }
    }
  }

  // ---------- POST-PIPELINE (altijd uitvoeren) ----------
  post {
    always {
      echo "🧹 Cleaning workspace..."
      cleanWs(deleteDirs: true, disableDeferredWipeout: true)
    }
  }
}
