import groovy.json.JsonOutput

pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
    timeout(time: 15, unit: 'MINUTES')
  }

  parameters {
    string(
      name: 'SOURCE_BRANCH',
      trim: true,
      description: 'Branch cần review (vd: feature/login)'
    )
    string(
      name: 'TARGET_BRANCH',
      defaultValue: 'main',
      trim: true,
      description: 'Base branch để so sánh'
    )
    string(
      name: 'PR_ID',
      defaultValue: 'manual',
      trim: true,
      description: 'PR ID (optional)'
    )
  }

  environment {
    GIT_CREDENTIAL = 'demo_github'
    GIT_REPO_URL   = 'https://github.com/levanhieu98/Fish-sauce.git'

    PROJECT_NAME  = 'Fish-sauce'
    REVIEW_MODE   = 'MANUAL_BRANCH_REVIEW'

    WEBHOOK_URL   = 'https://script.google.com/macros/s/AKfycbwKJ4Xh0v02OdTUbS96Ie-cvZno1INGrN8Ex7KtLEWrVm9LfjH1x1F9MO-lvHkeBIrQ/exec'

    MAX_DIFF_SIZE = '300000' // bytes
  }

  stages {

    /* =======================
     * 1. VALIDATE INPUT
     * ======================= */
    stage('Validate Input') {
      steps {
        script {
          if (!params.SOURCE_BRANCH) {
            error "❌ SOURCE_BRANCH is required"
          }
          if (params.SOURCE_BRANCH == params.TARGET_BRANCH) {
            error "❌ SOURCE_BRANCH must be different from TARGET_BRANCH"
          }
        }
      }
    }

    /* =======================
     * 2. CHECKOUT SOURCE
     * ======================= */
    stage('Checkout Source Branch') {
      steps {
        checkout([
          $class: 'GitSCM',
          branches: [[name: "refs/heads/${params.SOURCE_BRANCH}"]],
          userRemoteConfigs: [[
            url: env.GIT_REPO_URL,
            credentialsId: env.GIT_CREDENTIAL
          ]],
          extensions: [
            [$class: 'CleanBeforeCheckout'],
            [$class: 'CloneOption', noTags: true, shallow: false]
          ]
        ])
      }
    }

    /* =======================
     * 3. COLLECT DIFF
     * ======================= */
    stage('Collect Diff') {
      steps {
        sh """
          set -e
          git fetch origin ${params.TARGET_BRANCH}
          git diff origin/${params.TARGET_BRANCH}..HEAD > diff.txt
          wc -c diff.txt
        """
      }
    }

    /* =======================
     * 4. VALIDATE DIFF
     * ======================= */
    stage('Validate Diff') {
      steps {
        script {
          def diffSize = sh(
            script: "wc -c diff.txt | awk '{print \$1}'",
            returnStdout: true
          ).trim().toInteger()

          echo "📦 Diff size: ${diffSize} bytes"

          if (diffSize < 50) {
            error "❌ Diff quá nhỏ – không đủ dữ liệu review"
          }

          if (diffSize > env.MAX_DIFF_SIZE.toInteger()) {
            error "❌ Diff quá lớn – vượt giới hạn ${env.MAX_DIFF_SIZE} bytes"
          }

          env.DIFF_SIZE = diffSize.toString()
        }
      }
    }

    /* =======================
     * 5. SEND TO GEMINI (GAS)
     * ======================= */
    stage('Send to Gemini AI') {
      steps {
        script {

          sh "command -v base64 >/dev/null"

          def diffBase64 = sh(
            script: "cat diff.txt | base64 | tr -d '\\n'",
            returnStdout: true
          ).trim()

          def payload = [
            project     : env.PROJECT_NAME,
            mode        : env.REVIEW_MODE,
            pr_number   : params.PR_ID,
            source      : params.SOURCE_BRANCH,
            target      : params.TARGET_BRANCH,
            diff_size   : env.DIFF_SIZE,
            diff_base64 : diffBase64,
            commit      : sh(script: "git rev-parse HEAD", returnStdout: true).trim(),
            author      : sh(script: "git log -1 --pretty=format:'%an'", returnStdout: true).trim(),
            timestamp   : new Date().format("yyyy-MM-dd HH:mm:ss", TimeZone.getTimeZone('Asia/Ho_Chi_Minh'))
          ]

          writeFile(
            file: 'payload.json',
            text: JsonOutput.prettyPrint(JsonOutput.toJson(payload))
          )

          sh """
            curl -s -X POST "${env.WEBHOOK_URL}" \
              -H "Content-Type: application/json" \
              --data @payload.json
          """
        }
      }
    }
  }

  /* =======================
   * POST ACTIONS
   * ======================= */
  post {
    success {
      echo "✅ AI Code Review request sent successfully"
    }
    failure {
      echo "❌ Pipeline failed"
    }
    always {
      archiveArtifacts artifacts: 'diff.txt, payload.json', onlyIfSuccessful: false
      cleanWs()
    }
  }
}
