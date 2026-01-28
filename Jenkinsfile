import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        GIT_CREDENTIAL = 'demo_github'
        BASE_BRANCH    = 'main'
        WEBHOOK_URL    = 'https://script.google.com/macros/s/AKfycbwKJ4Xh0v02OdTUbS96Ie-cvZno1INGrN8Ex7KtLEWrVm9LfjH1x1F9MO-lvHkeBIrQ/exec'
        PROJECT_NAME   = 'Fish-sauce'
    }

    stages {

        /* =========================
           1. CHECKOUT SOURCE CODE
        ========================== */
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '**']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/levanhieu98/Fish-sauce.git',
                        credentialsId: env.GIT_CREDENTIAL
                    ]]
                ])
            }
        }

        /* =========================
           2. COLLECT DIFF (PR / MAIN)
        ========================== */
        stage('Collect Diff') {
            steps {
                sh '''
                  echo "================================"
                  echo " Collecting Git Diff"
                  echo "================================"

                  if [ -n "$CHANGE_ID" ]; then
                    echo "🟢 Pull Request detected"
                    echo "PR ID        : $CHANGE_ID"
                    echo "Source branch: $CHANGE_BRANCH"
                    echo "Target branch: $CHANGE_TARGET"

                    git fetch origin $CHANGE_TARGET
                    git diff origin/$CHANGE_TARGET...HEAD > diff.txt
                  else
                    echo "🟡 Direct push detected"

                    if git rev-parse HEAD~1 >/dev/null 2>&1; then
                      git diff HEAD~1 HEAD > diff.txt
                    else
                      git show HEAD > diff.txt
                    fi
                  fi

                  echo "--------------------------------"
                  echo "Diff preview:"
                  head -200 diff.txt
                  echo "--------------------------------"
                '''
            }
        }

        /* =========================
           3. SEND TO GEMINI (AI REVIEW)
        ========================== */
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
                        error "❌ Diff quá nhỏ hoặc rỗng – bỏ qua AI review"
                    }

                    def diffBase64 = sh(
                        script: "base64 diff.txt | tr -d '\\n'",
                        returnStdout: true
                    ).trim()

                    def payload = [
                        project        : PROJECT_NAME,
                        repo           : PROJECT_NAME,
                        commit         : commitHash,
                        author         : authorName,
                        diff_base64    : diffBase64,
                        diff_size      : diffSize,
                        is_pr          : env.CHANGE_ID ? true : false,
                        pr_id          : env.CHANGE_ID ?: '',
                        pr_branch      : env.CHANGE_BRANCH ?: '',
                        base_branch    : env.CHANGE_TARGET ?: BASE_BRANCH,
                        build_number   : env.BUILD_NUMBER,
                        build_url      : env.BUILD_URL
                    ]

                    writeFile file: 'payload.json', text: JsonOutput.prettyPrint(JsonOutput.toJson(payload))

                    sh '''
                      echo "================================"
                      echo " Sending payload to Gemini"
                      echo "================================"

                      curl -s -L -X POST "$WEBHOOK_URL" \
                        -H "Content-Type: application/json" \
                        -d @payload.json > response.json

                      echo "--------------------------------"
                      echo "Gemini response:"
                      cat response.json
                      echo "--------------------------------"
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
