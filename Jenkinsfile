import groovy.json.JsonOutput

pipeline {
  agent any

  /*********************************
   * 1. INPUT – REVIEWER NHẬP TAY
   *********************************/
  parameters {
    string(
      name: 'PR_ID',
      description: 'Pull Request ID (ví dụ: 123)'
    )
    string(
      name: 'SOURCE_BRANCH',
      description: 'PR source branch (ví dụ: feature/login)'
    )
    string(
      name: 'TARGET_BRANCH',
      defaultValue: 'main',
      description: 'Base branch (mặc định: main)'
    )
  }

  environment {
    GIT_CREDENTIAL = 'demo_github'
    PROJECT_NAME   = 'Fish-sauce'
    WEBHOOK_URL    = 'https://script.google.com/macros/s/AKfycbwKJ4Xh0v02OdTUbS96Ie-cvZno1INGrN8Ex7KtLEWrVm9LfjH1x1F9MO-lvHkeBIrQ/exec'
    MAX_DIFF_SIZE  = '300000'
  }

  stages {

    /*********************************
     * 2. VALIDATE INPUT
     *********************************/
    stage('Validate PR Input') {
      steps {
        script {
          if (!params.PR_ID?.trim()) {
            error "❌ PR_ID is required"
          }
          if (!params.SOURCE_BRANCH?.trim()) {
            error "❌ SOURCE_BRANCH is required"
          }

          echo "🔍 Manual PR Review"
          echo "PR #${params.PR_ID}: ${params.SOURCE_BRANCH} → ${params.TARGET_BRANCH}"
        }
      }
    }

    /*********************************
     * 3. CHECKOUT SOURCE BRANCH
     *********************************/
    stage('Checkout PR Branch') {
      steps {
        checkout([
          $class: 'GitSCM',
          branches: [[name: "refs/heads/${params.SOURCE_BRANCH}"]],
          userRemoteConfigs: [[
            url: 'https://github.com/levanhieu98/Fish-sauce.git',
            credentialsId: env.GIT_CREDENTIAL
          ]]
        ])

        sh 'git fetch --all'
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
     * 5. SEND TO AI (GEMINI)
     *********************************/
    stage('Send to Gemini AI') {
      steps {
        script {
          def diffSize = sh(
            script: "wc -c diff.txt | awk '{print \$1}'",
            returnStdout: true
          ).trim().toInteger()

          if (diffSize < 50) {
            error "❌ Diff quá nhỏ – không cần review"
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
            repo         : env.PROJECT_NAME,
            project      : env.PROJECT_NAME,
            mode         : "MANUAL_PR_REVIEW",
            pr_number    : params.PR_ID,
            source       : params.SOURCE_BRANCH,
            target       : params.TARGET_BRANCH,
            commit       : commitHash,
            author       : authorName,
            diff_size    : diffSize,
            diff_base64  : diffBase64,
            review_rule  : "security,performance,clean-code"
          ]

          writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

          sh '''
            echo "🚀 Sending PR diff to Gemini AI..."
            curl -s -X POST "$WEBHOOK_URL" \
              -H "Content-Type: application/json" \
              -d @payload.json > response.json

            echo "🤖 Gemini AI response:"
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ Manual AI PR Review completed"
    }
    failure {
      echo "❌ Manual AI PR Review failed"
    }
  }
}
