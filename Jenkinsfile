import groovy.json.JsonOutput

pipeline {
  agent any

  /*********************************
   * 1. INPUT – REVIEWER CHỈ CHỌN BASE
   *********************************/
  parameters {
    string(
      name: 'TARGET_BRANCH',
      defaultValue: 'main',
      description: 'Base branch để so sánh (ví dụ: main, develop)'
    )
    booleanParam(
      name: 'FORCE_REVIEW',
      defaultValue: false,
      description: 'Force review ngay cả khi diff nhỏ'
    )
  }

  environment {
    PROJECT_NAME  = 'Fish-sauce'
    GIT_CREDENTIAL = 'demo_github'
    WEBHOOK_URL   = 'https://script.google.com/macros/s/AKfycbwKJ4Xh0v02OdTUbS96Ie-cvZno1INGrN8Ex7KtLEWrVm9LfjH1x1F9MO-lvHkeBIrQ/exec'
    MAX_DIFF_SIZE = '300000'
  }

  stages {

    /*********************************
     * 2. CHECKOUT (HEAD = BRANCH ĐANG REVIEW)
     *********************************/
    stage('Checkout Source Branch') {
      steps {
        checkout scm
        sh 'git fetch --all --prune'
      }
    }

    /*********************************
     * 3. COLLECT METADATA
     *********************************/
    stage('Collect Context') {
      steps {
        script {
          env.SOURCE_BRANCH = sh(
            script: "git rev-parse --abbrev-ref HEAD",
            returnStdout: true
          ).trim()

          env.COMMIT_HASH = sh(
            script: "git rev-parse HEAD",
            returnStdout: true
          ).trim()

          env.AUTHOR = sh(
            script: "git log -1 --pretty=%an",
            returnStdout: true
          ).trim()

          echo "🔍 Reviewing branch: ${env.SOURCE_BRANCH}"
          echo "➡️ Target branch: ${params.TARGET_BRANCH}"
        }
      }
    }

    /*********************************
     * 4. COLLECT DIFF
     *********************************/
    stage('Collect Diff') {
      steps {
        sh """
          echo "📄 Collecting diff..."
          git fetch origin ${params.TARGET_BRANCH}

          git diff origin/${params.TARGET_BRANCH}...HEAD > diff.txt
          wc -c diff.txt
        """
      }
    }

    /*********************************
     * 5. VALIDATE DIFF
     *********************************/
    stage('Validate Diff') {
      steps {
        script {
          def diffSize = sh(
            script: "wc -c diff.txt | awk '{print \$1}'",
            returnStdout: true
          ).trim().toInteger()

          echo "📦 Diff size: ${diffSize} bytes"

          if (diffSize < 50 && !params.FORCE_REVIEW) {
            error "❌ Diff quá nhỏ – không cần AI review"
          }

          if (diffSize > env.MAX_DIFF_SIZE.toInteger()) {
            error "❌ Diff quá lớn (${diffSize} bytes)"
          }

          env.DIFF_SIZE = diffSize.toString()
        }
      }
    }

    /*********************************
     * 6. SEND TO GEMINI AI
     *********************************/
    stage('Send to Gemini AI') {
      steps {
        script {
          def diffBase64 = sh(
            script: "base64 diff.txt | tr -d '\\n'",
            returnStdout: true
          ).trim()

          def payload = [
            repo        : env.PROJECT_NAME,
            project     : env.PROJECT_NAME,
            mode        : "MANUAL_BRANCH_REVIEW",
            source      : env.SOURCE_BRANCH,
            target      : params.TARGET_BRANCH,
            commit      : env.COMMIT_HASH,
            author      : env.AUTHOR,
            diff_size   : env.DIFF_SIZE,
            diff_base64 : diffBase64,
            rules       : "security,performance,clean-code"
          ]

          writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

          sh '''
            echo "🚀 Sending diff to Gemini AI..."
            curl -s -X POST "$WEBHOOK_URL" \
              -H "Content-Type: application/json" \
              -d @payload.json > response.json

            echo "🤖 Gemini AI response received"
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
