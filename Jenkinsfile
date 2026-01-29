import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        // 👉 NÊN cấu hình trong Jenkins Credentials
        WEBHOOK_URL  = 'https://script.google.com/macros/s/AKfycby_GMrTUo2vCpRv3mzRfnW3CUsYhbQTf9p5jkRZqrG2VZpdodYsPWmn9r4NLZyW6I51/exec'

        PROJECT_NAME = 'Fish-sauce'
        BASE_BRANCH  = 'main'

        // Diff size limit (bytes)
        MIN_DIFF_SIZE = '50'
        MAX_DIFF_SIZE = '400000'
    }

    stages {

        /* =========================
           GUARD – PR ONLY
        ========================== */
        stage('Guard') {
            steps {
                script {
                    if (!env.CHANGE_ID) {
                        echo "⏭️ Skip: not a Pull Request build"
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
                  echo "PR ID         = $CHANGE_ID"
                  echo "PR branch     = $CHANGE_BRANCH"
                  echo "Base branch   = $CHANGE_TARGET"
                  echo "Commit        = $(git rev-parse HEAD)"
                '''
            }
        }

        /* =========================
           COLLECT PR DIFF
        ========================== */
        stage('Collect Diff') {
            steps {
                sh '''
                echo "🔍 Collecting PR diff..."

                # Fetch base branch into local ref
                git fetch origin ${CHANGE_TARGET}:refs/remotes/origin/${CHANGE_TARGET}

                # PR diff (same as GitHub)
                git diff refs/remotes/origin/${CHANGE_TARGET}...HEAD > diff.txt

                if [ ! -s diff.txt ]; then
                    echo "NO_DIFF=true" > .env
                fi

                echo "Diff size:"
                wc -c diff.txt || true
                '''
            }
        }

        /* =========================
           SEND TO GEMINI AI
        ========================== */
        stage('Send to Gemini') {
            steps {
                script {

                    // Skip nếu không có diff
                    if (fileExists('.env')) {
                        echo "⏭️ No code changes – skip AI review"
                        return
                    }

                    // Tính size diff
                    def diffSize = sh(
                        script: "wc -c diff.txt | awk '{print \$1}'",
                        returnStdout: true
                    ).trim().toInteger()

                    if (diffSize < env.MIN_DIFF_SIZE.toInteger()) {
                        echo "⏭️ Diff too small – skip AI review"
                        return
                    }

                    if (diffSize > env.MAX_DIFF_SIZE.toInteger()) {
                        echo "⚠️ Diff too large (${diffSize} bytes) – skip AI review"
                        return
                    }

                    // Build payload
                    def payload = [
                        project      : env.PROJECT_NAME,
                        repo         : env.JOB_NAME,
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

                    writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

                    // Gửi webhook (hardened)
                    sh '''
                      echo "🚀 Sending payload to Gemini AI..."
                      curl --connect-timeout 10 \
                           --max-time 30 \
                           --retry 3 \
                           --retry-delay 5 \
                           -s -L -X POST "$WEBHOOK_URL" \
                           -H "Content-Type: application/json" \
                           -d @payload.json \
                           > response.json || true

                      echo "Payload size: $(wc -c payload.json | awk '{print $1}') bytes"
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
