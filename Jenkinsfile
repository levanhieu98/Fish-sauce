import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        // 👉 NÊN cấu hình trong Jenkins Credentials
        WEBHOOK_URL  = 'https://script.google.com/macros/s/AKfycbwHHTCaFjSLxQK_MWi7ALHhoBXik9EDs-njFoJU6519OjeYXpl22dQsCZR3pTD-BLlZ/exec'

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
                        error("⏭️ Not a Pull Request build")
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
           COLLECT CHANGED FILES
        ========================== */
        stage('Collect Changed Files') {
            steps {
                sh '''
                  git fetch origin ${CHANGE_TARGET}:refs/remotes/origin/${CHANGE_TARGET}

                  git diff --name-only refs/remotes/origin/${CHANGE_TARGET}...HEAD > files.txt

                  echo "Changed files:"
                  cat files.txt
                '''
            }
        }

        /* =========================
           AI REVIEW PER FILE
        ========================== */
        stage('AI Review Per File') {
            steps {
                script {

                    def files = readFile('files.txt').trim().split('\n')

                    for (filePath in files) {

                        echo "🔍 Reviewing file: ${filePath}"

                        sh """
                          git diff refs/remotes/origin/${CHANGE_TARGET}...HEAD -- ${filePath} > diff_current.txt
                        """

                        def diffSize = sh(
                            script: "wc -c diff_current.txt | awk '{print \$1}'",
                            returnStdout: true
                        ).trim().toInteger()

                        if (diffSize < env.MIN_DIFF_SIZE.toInteger()) {
                            echo "⏭️ Skip ${filePath} (diff too small)"
                            continue
                        }

                        if (diffSize > env.MAX_DIFF_SIZE.toInteger()) {
                            echo "⚠️ Skip ${filePath} (diff too large)"
                            continue
                        }

                        def payload = [
                            project     : env.PROJECT_NAME,
                            repo        : env.JOB_NAME,
                            pr_id       : env.CHANGE_ID,
                            file        : filePath,
                            author      : sh(script: 'git log -1 --pretty=%an', returnStdout: true).trim(),
                            commit      : sh(script: 'git rev-parse HEAD', returnStdout: true).trim(),
                            diff_base64 : sh(script: "base64 diff_current.txt | tr -d '\\n'", returnStdout: true).trim(),
                            diff_size   : diffSize,
                            build_url   : env.BUILD_URL
                        ]

                        writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

                        sh '''
                          echo "🚀 Sending file diff to AI..."
                          curl -s -X POST "$WEBHOOK_URL" \
                               -H "Content-Type: application/json" \
                               -d @payload.json \
                               > response.json || true
                        '''

                        /* =========================
                           AI GENERATE TEST CASE
                        ========================== */
                        sh '''
                          echo "🧪 Generating test cases..."
                          curl -s -X POST "$WEBHOOK_URL?mode=testcase" \
                               -H "Content-Type: application/json" \
                               -d @payload.json \
                               > testcase.json || true
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ AI Review & Test Case generation completed"
        }
        always {
            archiveArtifacts artifacts: '*.txt,*.json', fingerprint: true
        }
    }
}
