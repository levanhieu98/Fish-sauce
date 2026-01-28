import groovy.json.JsonOutput

pipeline {
  agent any

  environment {
    GIT_CREDENTIAL = 'demo_github'
    WEBHOOK_URL    = 'https://script.google.com/macros/s/AKfycbwKJ4Xh0v02OdTUbS96Ie-cvZno1INGrN8Ex7KtLEWrVm9LfjH1x1F9MO-lvHkeBIrQ/exec'
    PROJECT_NAME   = 'Fish-sauce'
    MAX_DIFF_SIZE  = '300000'
  }

  stages {

    /* =========================
       1. CHECKOUT
    ========================== */
    stage('Checkout') {
      steps {
        checkout scm
        sh 'git fetch --all'
      }
    }

    /* =========================
       2. COLLECT DIFF
    ========================== */
    stage('Collect Diff') {
      steps {
        script {

          if (env.CHANGE_ID) {
            /* ===== PR MODE ===== */
            echo "🔍 PR MODE"
            echo "PR #${env.CHANGE_ID}: ${env.CHANGE_BRANCH} → ${env.CHANGE_TARGET}"

            sh """
              git fetch origin ${env.CHANGE_TARGET}
              git fetch origin ${env.CHANGE_BRANCH}

              git diff origin/${env.CHANGE_TARGET}...origin/${env.CHANGE_BRANCH} > diff.txt
            """

          } else {
            /* ===== PUSH / MERGE MODE ===== */
            echo "🔍 PUSH / MERGE MODE"

            sh '''
              if git rev-parse HEAD~1 >/dev/null 2>&1; then
                git diff HEAD~1 HEAD > diff.txt
              else
                git show HEAD > diff.txt
              fi
            '''
          }

          sh 'wc -c diff.txt'
        }
      }
    }

    /* =========================
       3. SEND TO GEMINI
    ========================== */
    stage('Send to Gemini AI') {
      steps {
        script {
          def diffSize = sh(
            script: "wc -c diff.txt | awk '{print \$1}'",
            returnStdout: true
          ).trim().toInteger()

          if (diffSize < 50) {
            error "❌ Diff rỗng – bỏ qua AI review"
          }

          if (diffSize > env.MAX_DIFF_SIZE.toInteger()) {
            error "❌ Diff quá lớn (${diffSize} bytes)"
          }

          def commitHash = sh(
            script: "git rev-parse HEAD",
            returnStdout: true
          ).trim()

          def authorName = sh(
            script: "git log -1 --pretty=%an",
            returnStdout: true
          ).trim()

          def diffBase64 = sh(
            script: "base64 diff.txt | tr -d '\\n'",
            returnStdout: true
          ).trim()

          def payload = [
            project     : env.PROJECT_NAME,
            mode        : env.CHANGE_ID ? "PR_REVIEW" : "POST_MERGE_REVIEW",
            pr_number   : env.CHANGE_ID ?: "",
            source      : env.CHANGE_BRANCH ?: "",
            target      : env.CHANGE_TARGET ?: "",
            commit      : commitHash,
            author      : authorName,
            diff_size   : diffSize,
            diff_base64 : diffBase64,
            review_rule : "security,performance,clean-code"
          ]

          writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

          sh '''
            echo "🚀 Sending diff to Gemini AI..."
            curl -s -X POST "$WEBHOOK_URL" \
              -H "Content-Type: application/json" \
              -d @payload.json > response.json

            echo "🤖 Gemini response:"
            cat response.json
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ AI Code Review completed successfully"
    }
    failure {
      echo "❌ AI Code Review failed"
    }
  }
}
