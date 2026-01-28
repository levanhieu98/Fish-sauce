import groovy.json.JsonOutput

pipeline {
  agent any

  parameters {
    string(name: 'SOURCE_BRANCH', description: 'Branch cần review (ví dụ: review_code)')
    string(name: 'TARGET_BRANCH', defaultValue: 'main', description: 'Base branch')
    string(name: 'PR_ID', defaultValue: 'manual', description: 'ID (optional)')
  }

  environment {
    GIT_CREDENTIAL = 'demo_github'
    PROJECT_NAME   = 'Fish-sauce'
    WEBHOOK_URL    = 'https://script.google.com/macros/s/AKfycbwKJ4Xh0v02OdTUbS96Ie-cvZno1INGrN8Ex7KtLEWrVm9LfjH1x1F9MO-lvHkeBIrQ/exec'
    MAX_DIFF_SIZE  = '300000'
  }

  stages {

    stage('Validate Input') {
      steps {
        script {
          if (!params.SOURCE_BRANCH?.trim()) {
            error "❌ SOURCE_BRANCH is required"
          }
          if (params.SOURCE_BRANCH == params.TARGET_BRANCH) {
            error "❌ SOURCE_BRANCH must be different from TARGET_BRANCH"
          }
        }
      }
    }

    stage('Checkout Source Branch') {
      steps {
        checkout([
          $class: 'GitSCM',
          branches: [[name: "refs/heads/${params.SOURCE_BRANCH}"]],
          userRemoteConfigs: [[
            url: 'https://github.com/levanhieu98/Fish-sauce.git',
            credentialsId: env.GIT_CREDENTIAL
          ]]
        ])

        sh 'git fetch origin'
      }
    }

    stage('Collect Diff') {
      steps {
        sh """
          git fetch origin ${params.TARGET_BRANCH}
          git diff origin/${params.TARGET_BRANCH}...HEAD > diff.txt
          wc -c diff.txt
        """
      }
    }

    stage('Send to Gemini AI') {
      steps {
        script {
          def diffSize = sh(
            script: "wc -c diff.txt | awk '{print \$1}'",
            returnStdout: true
          ).trim().toInteger()

          if (diffSize < 50) error "❌ Diff quá nhỏ"
          if (diffSize > env.MAX_DIFF_SIZE.toInteger()) error "❌ Diff quá lớn"

          def payload = [
            project     : env.PROJECT_NAME,
            mode        : "MANUAL_BRANCH_REVIEW",
            pr_number   : params.PR_ID,
            source      : params.SOURCE_BRANCH,
            target      : params.TARGET_BRANCH,
            diff_size   : diffSize,
            diff_base64 : sh(script: "base64 diff.txt | tr -d '\\n'", returnStdout: true).trim()
          ]

          writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

          sh """
            curl -s -X POST "$WEBHOOK_URL" \
              -H "Content-Type: application/json" \
              -d @payload.json
          """
        }
      }
    }
  }
}
