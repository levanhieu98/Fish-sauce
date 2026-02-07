import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        WEBHOOK_URL  = 'https://script.google.com/macros/s/AKfycbx0tGnwgnVkiPwAsCDpp7UxaBzREFdOBj0Q4vULTrXL8I0FQOQuFcaZIIzeTwtRsEyR/exec'

        PROJECT_NAME  = 'Event-Laravel'

        BASE_BRANCH   = 'main'      // nhánh gốc để so sánh
        REVIEW_BRANCH = 'review'    // nhánh được merge vào để review

        MIN_DIFF_SIZE = '50'
        MAX_DIFF_SIZE = '400000'
    }

    stages {

        /* =========================
           GUARD – REVIEW BRANCH ONLY
        ========================== */
        stage('Guard Review Branch') {
            steps {
                script {
                    def branch = env.BRANCH_NAME ?: sh(
                        script: 'git rev-parse --abbrev-ref HEAD',
                        returnStdout: true
                    ).trim()

                    if (branch != env.REVIEW_BRANCH) {
                        echo "⏭️ Skip: ${branch} is not review branch"
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
                  echo "Project       = ${PROJECT_NAME}"
                  echo "Review branch = $(git rev-parse --abbrev-ref HEAD)"
                  echo "Base branch   = ${BASE_BRANCH}"
                  echo "HEAD commit   = $(git rev-parse HEAD)"
                '''
            }
        }

        /* =========================
           CALCULATE DIFF BASE
        ========================== */
        stage('Calculate Diff Base') {
            steps {
                sh '''
                  git fetch origin ${BASE_BRANCH}

                  BASE_COMMIT=$(git merge-base origin/${BASE_BRANCH} HEAD)

                  echo "BASE_COMMIT=${BASE_COMMIT}" > diff_base.env
                  echo "Diff from ${BASE_COMMIT} -> HEAD"
                '''
            }
        }

        /* =========================
           COLLECT CHANGED FILES
        ========================== */
        stage('Collect Changed Files') {
            steps {
                sh '''
                  BASE_COMMIT=$(cat diff_base.env | cut -d= -f2)

                    git diff $BASE_COMMIT HEAD --name-only \
                    | grep -E '^(app|routes|database|resources)/' \
                    > files.txt || true


                  if [ ! -s files.txt ]; then
                    echo "⏭️ No relevant files changed"
                    exit 0
                  fi

                  echo "Files to review:"
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
                    def baseCommit = sh(
                        script: "cat diff_base.env | cut -d= -f2",
                        returnStdout: true
                    ).trim()

                    def baseCommit = sh(
                        script: "grep BASE_COMMIT diff_base.env | cut -d= -f2",
                        returnStdout: true
                    ).trim()

                    def files = readFile('files.txt').trim().split('\n')

                    for (filePath in files) {

                        echo "🔍 Reviewing ${filePath}"

                        sh """
                          git diff ${baseCommit} HEAD -- ${filePath} > diff_current.txt
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
                            project      : env.PROJECT_NAME,
                            repo         : env.JOB_NAME,
                            pr_branch    : env.REVIEW_BRANCH,
                            base_branch  : env.BASE_BRANCH,
                            base_commit  : baseCommit,
                            head_commit  : sh(script: 'git rev-parse HEAD', returnStdout: true).trim(),
                            file         : filePath,
                            author       : sh(script: 'git log -1 --pretty=%an', returnStdout: true).trim(),
                            diff_size    : diffSize,
                            diff_base64  : sh(script: "base64 diff_current.txt | tr -d '\\n'", returnStdout: true).trim(),
                            build_url    : env.BUILD_URL
                        ]

                        writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

                        sh '''
                          echo "🚀 AI Code Review"
                          curl -s -L -X POST "$WEBHOOK_URL" \
                               -H "Content-Type: application/json" \
                               -d @payload.json > response.json || true
                        '''

                        sh '''
                          echo "🧪 AI Generate Test Cases"
                          curl -s -L -X POST "$WEBHOOK_URL?mode=testcase" \
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
            echo "✅ AI Review completed successfully"
        }
        always {
            archiveArtifacts artifacts: '*.txt,*.json,*.env', fingerprint: true
        }
    }
}
