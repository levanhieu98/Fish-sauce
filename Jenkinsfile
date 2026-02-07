import groovy.json.JsonOutput

pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }

    environment {
        WEBHOOK_URL  = 'https://script.google.com/macros/s/AKfycbx0tGnwgnVkiPwAsCDpp7UxaBzREFdOBj0Q4vULTrXL8I0FQOQuFcaZIIzeTwtRsEyR/exec'

        PROJECT_NAME  = 'Event-Laravel'

        BASE_BRANCH   = 'main'
        REVIEW_BRANCH = 'review'

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
                    if (env.BRANCH_NAME != env.REVIEW_BRANCH) {
                        echo "⏭️ Skip: ${env.BRANCH_NAME} is not review branch"
                        currentBuild.result = 'NOT_BUILT'
                        error("Not review branch")
                    }
                    echo "✅ Review branch detected: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Debug Context') {
            steps {
                sh '''
                  echo "Project       = ${PROJECT_NAME}"
                  echo "Review branch = ${BRANCH_NAME}"
                  echo "Base branch   = ${BASE_BRANCH}"
                  echo "HEAD commit   = $(git rev-parse HEAD)"
                '''
            }
        }

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

        stage('Collect Changed Files') {
            steps {
                sh '''
                  BASE_COMMIT=$(cut -d= -f2 diff_base.env)

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

        stage('AI Review Per File') {
            steps {
                script {
                    if (!fileExists('files.txt')) {
                        echo "⏭️ No files to review"
                        return
                    }

                    def baseCommit = sh(
                        script: "cut -d= -f2 diff_base.env",
                        returnStdout: true
                    ).trim()

                    def files = readFile('files.txt').trim().split('\n')

                    for (filePath in files) {

                        echo "🔍 Reviewing ${filePath}"

                        sh "git diff ${baseCommit} HEAD -- ${filePath} > diff_current.txt"

                        def diffSize = sh(
                            script: "wc -c diff_current.txt | awk '{print \$1}'",
                            returnStdout: true
                        ).trim().toInteger()

                        if (diffSize < env.MIN_DIFF_SIZE.toInteger()) continue
                        if (diffSize > env.MAX_DIFF_SIZE.toInteger()) continue

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
                          curl -s -L -X POST "$WEBHOOK_URL" \
                            -H "Content-Type: application/json" \
                            -d @payload.json > response.json || true
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '*.txt,*.json,*.env', fingerprint: true
        }
    }
}
