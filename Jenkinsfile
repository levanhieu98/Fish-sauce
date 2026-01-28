import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        WEBHOOK_URL  = 'https://script.google.com/macros/s/AKfycbwKJ4Xh0v02OdTUbS96Ie-cvZno1INGrN8Ex7KtLEWrVm9LfjH1x1F9MO-lvHkeBIrQ/exec'
        PROJECT_NAME = 'Fish-sauce'
        BASE_BRANCH  = 'main'
    }

    stages {

        /* =========================
           DEBUG – KIỂM TRA NGỮ CẢNH
        ========================== */
        stage('Debug Context') {
            steps {
                sh '''
                  echo "================================"
                  echo "BRANCH_NAME     = $BRANCH_NAME"
                  echo "CHANGE_ID       = $CHANGE_ID"
                  echo "CHANGE_BRANCH   = $CHANGE_BRANCH"
                  echo "CHANGE_TARGET   = $CHANGE_TARGET"
                  echo "HEAD commit:"
                  git log -1 --oneline
                  echo "================================"
                '''
            }
        }

        /* =========================
           COLLECT DIFF (PR / PUSH)
        ========================== */
        stage('Collect Diff') {
            steps {
                sh '''
                  echo "Collecting git diff..."

                  if [ -n "$CHANGE_ID" ]; then
                    echo "🟢 Pull Request detected"
                    echo "Base branch: $CHANGE_TARGET"

                    # 🔥 BẮT BUỘC fetch base branch
                    git fetch origin $CHANGE_TARGET

                    # Diff đúng PR (giống GitHub)
                    git diff origin/$CHANGE_TARGET...HEAD > diff.txt
                  else
                    echo "🟡 Direct push detected"

                    if git rev-parse HEAD~1 >/dev/null 2>&1; then
                      git diff HEAD~1 HEAD > diff.txt
                    else
                      git show HEAD > diff.txt
                    fi
                  fi

                  echo "---- Diff preview ----"
                  head -200 diff.txt
                '''
            }
        }

        /* =========================
           SEND TO GEMINI (AI REVIEW)
        ========================== */
        stage('Send to Gemini') {
            steps {
                script {
                    def commitHash = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    def authorName = sh(
                        script: 'git log -1 --pretty=%an',
                        returnStdout: true
                    ).trim()

                    def diffSize = sh(
                        script: "wc -c diff.txt | awk '{print \$1}'",
                        returnStdout: true
                    ).trim()

                    if (diffSize.toInteger() < 50) {
                        error "❌ Diff quá nhỏ hoặc rỗng – không gửi AI review"
                    }

                    def diffBase64 = sh(
                        script: "base64 diff.txt | tr -d '\\n'",
                        returnStdout: true
                    ).trim()

                    def payload = [
                        project       : PROJECT_NAME,
                        repo          : PROJECT_NAME,
                        commit        : commitHash,
                        author        : authorName,
                        diff_base64   : diffBase64,
                        diff_size     : diffSize,
                        is_pr         : env.CHANGE_ID ? true : false,
                        pr_id         : env.CHANGE_ID ?: '',
                        pr_branch     : env.CHANGE_BRANCH ?: '',
                        base_branch   : env.CHANGE_TARGET ?: BASE_BRANCH,
                        build_number  : env.BUILD_NUMBER,
                        build_url     : env.BUILD_URL
                    ]

                    writeFile file: 'payload.json',
                              text: JsonOutput.toJson(payload)

                    sh '''
                      echo "---- Sending payload to Gemini ----"

                      curl -s -L -X POST "$WEBHOOK_URL" \
                        -H "Content-Type: application/json; charset=utf-8" \
                        --data-binary @payload.json \
                        > response.json

                      echo "---- Gemini response ----"
                      cat response.json
                    '''

                }
            }
        }
    }

    post {
        success {
            echo "✅ AI Code Review pipeline completed successfully"
        }
        failure {
            echo "❌ AI Code Review pipeline failed"
        }
    }
}
