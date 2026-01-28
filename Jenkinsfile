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
           GUARD – CHỈ CHẠY PR BUILD
        ========================== */
        stage('Guard') {
            steps {
                script {
                    if (!env.CHANGE_ID) {
                        echo "⏭️ Skip: Branch indexing or normal branch build"
                        currentBuild.result = 'NOT_BUILT'
                        error("Not a PR build")
                    }
                }
            }
        }

        /* =========================
           DEBUG CONTEXT
        ========================== */
        stage('Debug Context') {
            steps {
                sh '''
                  echo "PR ID           = $CHANGE_ID"
                  echo "PR branch       = $CHANGE_BRANCH"
                  echo "Target branch   = $CHANGE_TARGET"
                  git log -1 --oneline
                '''
            }
        }

        /* =========================
           COLLECT DIFF (PR ONLY)
        ========================== */
        stage('Collect Diff') {
          steps {
              sh '''
                echo "Collecting PR diff..."

                BASE_REF="base/${CHANGE_TARGET}"

                # Fetch base branch vào local ref riêng
                git fetch origin ${CHANGE_TARGET}:refs/remotes/${BASE_REF}

                # Diff giống GitHub PR
                git diff ${BASE_REF}...HEAD > diff.txt

                if [ ! -s diff.txt ]; then
                  echo "⏭️ No code changes in PR – skip AI review"
                  exit 0
                fi

                echo "---- Diff preview ----"
                head -200 diff.txt
              '''
          }
      }
        /* =========================
           SEND TO GEMINI
        ========================== */
        stage('Send to Gemini') {
            steps {
                script {
                    def diffSize = sh(
                        script: "wc -c diff.txt | awk '{print \$1}'",
                        returnStdout: true
                    ).trim()

                    if (diffSize.toInteger() < 50) {
                        echo "⏭️ Diff too small – skip"
                        return
                    }

                    def payload = [
                        project      : PROJECT_NAME,
                        repo         : PROJECT_NAME,
                        commit       : sh(script: 'git rev-parse HEAD', returnStdout: true).trim(),
                        author       : sh(script: 'git log -1 --pretty=%an', returnStdout: true).trim(),
                        diff_base64  : sh(script: "base64 diff.txt | tr -d '\\n'", returnStdout: true).trim(),
                        diff_size    : diffSize,
                        pr_id        : env.CHANGE_ID,
                        pr_branch    : env.CHANGE_BRANCH,
                        base_branch  : env.CHANGE_TARGET,
                        build_number : env.BUILD_NUMBER,
                        build_url    : env.BUILD_URL
                    ]

                    writeFile file: 'payload.json',
                              text: JsonOutput.toJson(payload)

                    sh '''
                      curl -s -L -X POST "$WEBHOOK_URL" \
                        -H "Content-Type: application/json" \
                        --data-binary @payload.json
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ AI Code Review completed"
        }
        failure {
            echo "❌ Pipeline failed"
        }
    }
}
