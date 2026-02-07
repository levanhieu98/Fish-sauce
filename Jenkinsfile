import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        WEBHOOK_URL  = 'https://script.google.com/macros/s/AKfycbx0tGnwgnVkiPwAsCDpp7UxaBzREFdOBj0Q4vULTrXL8I0FQOQuFcaZIIzeTwtRsEyR/exec'

        PROJECT_NAME = 'Event-Laravel'
        BASE_BRANCH  = 'main'          // nhánh để so diff

        REVIEW_BRANCH = 'review'  // nhánh trigger review

        MIN_DIFF_SIZE = '50'
        MAX_DIFF_SIZE = '400000'
    }

    stages {

        /* =========================
           GUARD – REVIEW BRANCH ONLY
        ========================== */
        stage('Guard') {
            steps {
                script {
                    def branch = env.BRANCH_NAME ?: sh(
                        script: 'git rev-parse --abbrev-ref HEAD',
                        returnStdout: true
                    ).trim()

                    if (branch != env.REVIEW_BRANCH) {
                        echo "⏭️ Skip: branch ${branch} not for review"
                        currentBuild.result = 'NOT_BUILT'
                        error("Not review branch")
                    }

                    echo "✅ Review branch detected: ${branch}"
                }
            }
        }

        /* =========================
           DEBUG CONTEXT
        ========================== */
        stage('Debug Context') {
            steps {
                sh '''
                  echo "Review branch = $(git rev-parse --abbrev-ref HEAD)"
                  echo "Base branch   = ${BASE_BRANCH}"
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
                  git fetch origin ${BASE_BRANCH}:refs/remotes/origin/${BASE_BRANCH}

                  git diff --name-only refs/remotes/origin/${BASE_BRANCH}...HEAD > files.txt

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
                          git diff refs/remotes/origin/${BASE_BRANCH}...HEAD -- ${filePath} > diff_current.txt
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
                            review_branch : env.REVIEW_BRANCH,
                            base_branch : env.BASE_BRANCH,
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
                               -d @payload.json > response.json || true
                        '''

                        sh '''
                          echo "🧪 Generating test cases..."
                          curl -s -X POST "$WEBHOOK_URL?mode=testcase" \
                               -H "Content-Type: application/json" \
                               -d @payload.json > testcase.json || true
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
