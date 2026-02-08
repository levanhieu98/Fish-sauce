import groovy.json.JsonOutput

pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timeout(time: 10, unit: 'MINUTES')
    }

    environment {
        WEBHOOK_URL  = 'https://script.google.com/macros/s/AKfycbyWwDZj9H3g5HS7SFY7EmdiWCSFEozCiMi_VDZJAa0RtksuLfCTBxLMwprnoFqnc499/exec'

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
                    def branch = env.BRANCH_NAME ?: sh(
                        script: 'git rev-parse --abbrev-ref HEAD',
                        returnStdout: true
                    ).trim()

                    if (branch != env.REVIEW_BRANCH) {
                        echo "⏭️ Skip: branch ${branch} not for review"
                        return
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
           CALCULATE MERGE BASE
        ========================== */
        stage('Calculate Merge Base') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'demo_github',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_PASS'
                    )
                ]) {
                    sh '''
                      git fetch https://${GIT_USER}:${GIT_PASS}@github.com/levanhieu98/Fish-sauce.git ${BASE_BRANCH}

                      BASE_COMMIT=$(git merge-base origin/${BASE_BRANCH} HEAD)

                      if [ -z "$BASE_COMMIT" ]; then
                        echo "❌ Cannot calculate merge-base"
                        exit 1
                      fi

                      echo "BASE_COMMIT=${BASE_COMMIT}" > diff_base.env
                      echo "Merge-base commit: ${BASE_COMMIT}"
                    '''
                }
            }
        }

        /* =========================
           COLLECT CHANGED FILES
        ========================== */
        stage('Collect Changed Files') {
            steps {
                sh '''
                  BASE_COMMIT=$(cut -d= -f2 diff_base.env)

                  git diff ${BASE_COMMIT} HEAD --name-only \
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
                    if (!fileExists('files.txt')) return

                    def raw = readFile('files.txt').trim()
                    if (!raw) return

                    def baseCommit = sh(
                        script: "cut -d= -f2 diff_base.env",
                        returnStdout: true
                    ).trim()

                    def headCommit = sh(
                        script: "git rev-parse HEAD",
                        returnStdout: true
                    ).trim()

                    def files = raw.split('\n')

                    for (filePath in files) {

                        echo "🔍 Reviewing ${filePath}"

                        sh """
                          git diff ${baseCommit}..${headCommit} -- ${filePath} > diff_current.txt
                        """

                        def diffSize = sh(
                            script: "wc -c diff_current.txt | awk '{print \$1}'",
                            returnStdout: true
                        ).trim().toInteger()

                        if (diffSize < env.MIN_DIFF_SIZE.toInteger()) continue
                        if (diffSize > env.MAX_DIFF_SIZE.toInteger()) continue

                        /* ---------- Detect file type ---------- */
                        def fileType = "other"
                        if (filePath.contains("/Controllers/")) fileType = "controller"
                        else if (filePath.contains("/Models/")) fileType = "model"
                        else if (filePath.contains("/Services/")) fileType = "service"
                        else if (filePath.contains("/Requests/")) fileType = "request"
                        else if (filePath.contains("/Migrations/")) fileType = "migration"

                        /* ---------- Authors ---------- */
                        def authorsRaw = sh(
                            script: """
                              git log ${baseCommit}..${headCommit} -- ${filePath} --pretty=%an | sort | uniq
                            """,
                            returnStdout: true
                        ).trim()

                        def authors = authorsRaw ? authorsRaw.split('\n') : []

                        /* ---------- Diff hash ---------- */
                        def diffHash = sh(
                            script: "sha256sum diff_current.txt | awk '{print \$1}'",
                            returnStdout: true
                        ).trim()

                        /* =========================
                           PAYLOAD (STANDARDIZED)
                        ========================== */
                        def payload = [
                            meta: [
                                project   : env.PROJECT_NAME,
                                repo      : env.JOB_NAME,
                                build_id  : "${env.JOB_NAME}#${env.BUILD_NUMBER}",
                                build_url : env.BUILD_URL,
                                timestamp : new Date().format("yyyy-MM-dd'T'HH:mm:ss'Z'")
                            ],

                            context: [
                                language     : "php",
                                framework    : "laravel",
                                php_version  : "8.2",
                                architecture : "mvc",
                                environment  : "staging",
                                base_branch  : env.BASE_BRANCH,
                                review_branch: env.REVIEW_BRANCH
                            ],

                            changeset: [
                                file         : filePath,
                                file_type   : fileType,
                                change_type : "modify",
                                authors     : authors,
                                base_commit : baseCommit,
                                head_commit : headCommit,
                                diff_size   : diffSize,
                                diff_hash   : diffHash,
                                diff        : [
                                    encoding: "base64",
                                    content : sh(
                                        script: "base64 -w 0 diff_current.txt",
                                        returnStdout: true
                                    ).trim()
                                ]
                            ],

                            intent: [
                                review_type   : "code_review",
                                focus         : [
                                    "bug",
                                    "security",
                                    "performance",
                                    "laravel_best_practice"
                                ],
                                severity_level: ["critical", "major", "minor"],
                                output_format : "markdown",
                                max_comments  : 10
                            ]
                        ]

                        writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

                        /* ---------- AI Code Review ---------- */
                        sh '''
                          echo "🚀 AI Code Review"
                          for i in 1 2 3; do
                            curl -s -L -X POST "$WEBHOOK_URL" \
                              -H "Content-Type: application/json" \
                              -d @payload.json && break
                            sleep 2
                          done
                        '''

                        /* ---------- AI Test Case ---------- */
                        sh '''
                          echo "🧪 AI Generate Test Cases"
                          for i in 1 2 3; do
                            curl -s -L -X POST "$WEBHOOK_URL?mode=testcase" \
                              -H "Content-Type: application/json" \
                              -d @payload.json && break
                            sleep 2
                          done
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ AI Review (merge-base + standardized payload) completed"
        }
        always {
            archiveArtifacts artifacts: '*.txt,*.json,*.env', fingerprint: true
        }
    }
}
