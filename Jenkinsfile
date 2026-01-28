import groovy.json.JsonOutput

pipeline {
  agent any

  environment {
    GIT_CREDENTIAL = 'demo_github'
    WEBHOOK_URL    = 'https://script.google.com/macros/s/AKfycbwKJ4Xh0v02OdTUbS96Ie-cvZno1INGrN8Ex7KtLEWrVm9LfjH1x1F9MO-lvHkeBIrQ/exec'
    PROJECT_NAME   = 'Fish-sauce'
    MAX_DIFF_SIZE  = '300000'
  }

  options {
    skipDefaultCheckout()
  }

  stages {

    /* =========================
       1. CHECKOUT
    ========================== */
    stage('Checkout') {
      steps {
        deleteDir()          // 🧹 dọn workspace tránh dính diff cũ
        checkout scm
        sh 'git fetch --all'
      }
    }

    /* =========================
       2. COLLECT DIFF (PR ONLY)
    ========================== */
    stage('Collect Diff') {
      steps {
        script {

            /* =========================================
             ❌ KHÔNG PR → DỪNG PIPELINE TẠI ĐÂY
             ========================================= */
          if (!env.CHANGE_ID) {
            error """
            ❌ This pipeline is configured for Pull Request only.

            ℹ️ Push / Merge logic is intentionally DISABLED.
            ℹ️ See commented code below for reference.
            """
          }

          /* ===== PR MODE ===== */
          echo "🔍 PR MODE"
          echo "PR #${env.CHANGE_ID}: ${env.CHANGE_BRANCH} → ${env.CHANGE_TARGET}"

          sh """
            git fetch origin ${env.CHANGE_TARGET}
            git fetch origin ${env.CHANGE_BRANCH}

            git diff origin/${env.CHANGE_TARGET}...origin/${env.CHANGE_BRANCH} > diff.txt
          """

          sh 'wc -c diff.txt'

          /* =====================================================
             📝 PUSH / MERGE MODE (REFERENCE ONLY – NOT USED)
             
             Mục đích:
             - Dùng cho demo
             - Dùng giải thích kiến trúc
             - KHÔNG kích hoạt trong pipeline hiện tại

             if (!env.CHANGE_ID) {
               echo "🔍 PUSH / MERGE MODE"

               sh '''
                 if git rev-parse HEAD~1 >/dev/null 2>&1; then
                   git diff HEAD~1 HEAD > diff.txt
                 else
                   git show HEAD > diff.txt
                 fi
               '''
             }
             ===================================================== */
        }
      }
    }

    /* =========================
       3. SEND TO GEMINI (PR ONLY)
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

          def payload = [
            repo         : PROJECT_NAME,
            project      : env.PROJECT_NAME,
            mode         : "PR_REVIEW",
            pr_number    : env.CHANGE_ID,
            source       : env.CHANGE_BRANCH,
            target       : env.CHANGE_TARGET,
            commit       : sh(script: "git rev-parse HEAD", returnStdout: true).trim(),
            author       : sh(script: "git log -1 --pretty=%an", returnStdout: true).trim(),
            diff_size    : diffSize,
            diff_base64  : sh(script: "base64 diff.txt | tr -d '\\n'", returnStdout: true).trim(),
            review_rule  : "security,performance,clean-code"
          ]

          writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

          sh '''
            echo "🚀 Sending diff to Gemini AI..."
            curl -s -X POST "$WEBHOOK_URL" \
              -H "Content-Type: application/json" \
              -d @payload.json > response.json

            echo "🤖 Gemini response:"
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ AI Code Review (PR only) completed successfully"
    }
    failure {
      echo "❌ AI Code Review failed or skipped (non-PR)"
    }
  }
}
