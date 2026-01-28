import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        WEBHOOK_URL  = 'https://script.google.com/macros/s/AKfycbwKJ4Xh0v02OdTUbS96Ie-cvZno1INGrN8Ex7KtLEWrVm9LfjH1x1F9MO-lvHkeBIrQ/exec'
        PROJECT_NAME = 'Fish-sauce'
        BASE_BRANCH  = 'main'
    }

    stages {

        stage('Debug Context') {
            steps {
                sh '''
                  echo "BRANCH_NAME     = $BRANCH_NAME"
                  echo "CHANGE_ID       = $CHANGE_ID"
                  echo "CHANGE_BRANCH   = $CHANGE_BRANCH"
                  echo "CHANGE_TARGET   = $CHANGE_TARGET"
                  git log -1 --oneline
                '''
            }
        }

        stage('Collect Diff') {
            steps {
                sh '''
                  echo "Collecting diff..."

                  if [ -n "$CHANGE_ID" ]; then
                    echo "PR detected"
                    git diff origin/$CHANGE_TARGET...HEAD > diff.txt
                  else
                    echo "Push detected"
                    git diff HEAD~1 HEAD > diff.txt
                  fi

                  head -200 diff.txt
                '''
            }
        }

        stage('Send to Gemini') {
            steps {
                script {
                    def commitHash = sh(
                        script: "git rev-parse HEAD",
                        returnStdout: true
                    ).trim()

                    def authorName = sh(
                        script: "git log -1 --pretty=%an",
                        returnStdout: true
                    ).trim()

                    def diffSize = sh(
                        script: "wc -c diff.txt | awk '{print \$1}'",
                        returnStdout: true
                    ).trim()

                    if (diffSize.toInteger() < 50) {
                        error "❌ Diff quá nhỏ hoặc rỗng – không gửi AI review"
                    }

                    // Encode base64 để tránh lỗi ký tự
                    def diffBase64 = sh(
                        script: "base64 diff.txt | tr -d '\\n'",
                        returnStdout: true
                    ).trim()

                    def payload = [
                        repo         : PROJECT_NAME, 
                        project      : PROJECT_NAME,
                        commit       : commitHash,
                        author       : authorName,
                        diff_base64  : diffBase64,
                        diff_size    : diffSize
                    ]

                    writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

                    sh '''
                        echo "--- Sending payload to Gemini ---"
                        curl -s -L -X POST "$WEBHOOK_URL" \
                          -H "Content-Type: application/json" \
                          -d @payload.json > response.json

                        echo "--- Response ---"
                        cat response.json
                    '''
                }
            }
        }
    }
}
