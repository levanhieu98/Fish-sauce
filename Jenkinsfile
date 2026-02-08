import groovy.json.JsonOutput

pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timeout(time: 15, unit: 'MINUTES')
        timestamps()
    }

    environment {
        WEBHOOK_URL = 'https://script.google.com/macros/s/AKfycbwXLKMQP-EyY3dtYzOkKC6nmIp6HiI10y2B3VJfS0VispBuXVyFzQGBx5-D4ebNT85o/exec'

        PROJECT_NAME  = 'Event-Laravel'
        BASE_BRANCH   = 'main'
        REVIEW_BRANCH = 'review'

        MIN_DIFF_SIZE = '50'
        MAX_DIFF_SIZE = '30000'
    }

    stages {

        /* ===============================
           GUARD BRANCH
        =============================== */
        stage('Guard Review Branch') {
            steps {
                script {
                    def branch = env.BRANCH_NAME ?: sh(
                        script: 'git rev-parse --abbrev-ref HEAD',
                        returnStdout: true
                    ).trim()

                    if (branch != env.REVIEW_BRANCH) {
                        currentBuild.result = 'NOT_BUILT'
                        error("Skip build: branch ${branch} is not ${env.REVIEW_BRANCH}")
                    }

                    echo "✔ Review branch detected: ${branch}"
                }
            }
        }

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

        /* ===============================
           CALCULATE MERGE BASE
        =============================== */
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

                      BASE_COMMIT=$(git merge-base FETCH_HEAD HEAD)

                      if [ -z "$BASE_COMMIT" ]; then
                        echo "❌ Cannot calculate merge-base"
                        exit 1
                      fi

                      echo "BASE_COMMIT=${BASE_COMMIT}" > diff_base.env
                      echo "Merge-base: ${BASE_COMMIT}"
                    '''
                }
            }
        }

        /* ===============================
           COLLECT FILES
        =============================== */
        stage('Collect Changed Files') {
            steps {
                sh '''
                  BASE_COMMIT=$(cut -d= -f2 diff_base.env)

                  git diff --name-status ${BASE_COMMIT} HEAD \
                    | grep -E '^(A|M|R|D)[[:space:]]+(app|routes|database|resources)/' \
                    > files_status.txt || true

                  if [ ! -s files_status.txt ]; then
                    echo "NO_CHANGES=true" > no_changes.env
                    echo "No relevant files changed"
                  fi

                  cat files_status.txt || true
                '''
            }
        }

        /* ===============================
           AI REVIEW PER FILE
        =============================== */
        stage('AI Review Per File') {
            when {
                expression { !fileExists('no_changes.env') }
            }
            steps {
                script {

                    def baseCommit = sh(
                        script: "cut -d= -f2 diff_base.env",
                        returnStdout: true
                    ).trim()

                    def headCommit = sh(
                        script: "git rev-parse HEAD",
                        returnStdout: true
                    ).trim()

                    def lines = readFile('files_status.txt').trim().split('\n')

                    for (line in lines) {

                        def parts = line.tokenize()
                        def changeTypeCode = parts[0]

                        def oldPath = parts.size() > 2 ? parts[1] : null
                        def filePath = parts.size() > 2 ? parts[2] : parts[1]

                        def changeType = [
                            'A': 'add',
                            'M': 'modify',
                            'D': 'delete',
                            'R': 'rename'
                        ][changeTypeCode] ?: 'modify'

                        echo "🔍 Reviewing ${filePath} (${changeType})"

                        /* === FULL CONTEXT DIFF === */
                        sh """
                          git diff ${baseCommit}..${headCommit} --unified=3 -- ${filePath} \
                            | head -c ${MAX_DIFF_SIZE} > diff_current.txt
                        """

                        def diffSize = sh(
                            script: "wc -c diff_current.txt | awk '{print \$1}'",
                            returnStdout: true
                        ).trim().toInteger()

                        if (diffSize < env.MIN_DIFF_SIZE.toInteger()) {
                            echo "⏭ Skip small diff (${diffSize} bytes)"
                            continue
                        }

                        /* === FILE TYPE === */
                        def fileType = "other"
                        if (filePath.contains("/Controllers/")) fileType = "controller"
                        else if (filePath.contains("/Models/")) fileType = "model"
                        else if (filePath.contains("/Services/")) fileType = "service"
                        else if (filePath.contains("/Requests/")) fileType = "request"
                        else if (filePath.contains("/Jobs/")) fileType = "job"
                        else if (filePath.contains("/Policies/")) fileType = "policy"
                        else if (filePath.contains("/Observers/")) fileType = "observer"
                        else if (filePath.contains("/Events/")) fileType = "event"
                        else if (filePath.contains("/Listeners/")) fileType = "listener"
                        else if (filePath.contains("/Migrations/")) fileType = "migration"

                        def authorsRaw = sh(
                            script: "git log ${baseCommit}..${headCommit} -- ${filePath} --pretty=%an | sort | uniq",
                            returnStdout: true
                        ).trim()

                        def authors = authorsRaw ? authorsRaw.split('\n') : []

                        def diffHash = sh(
                            script: "sha256sum diff_current.txt | awk '{print \$1}'",
                            returnStdout: true
                        ).trim()

                        def payload = [
                            meta: [
                                review_id : "${env.BUILD_TAG}-${diffHash.take(8)}",
                                project   : env.PROJECT_NAME,
                                build_id  : "${env.JOB_NAME}#${env.BUILD_NUMBER}",
                                build_url : env.BUILD_URL
                            ],
                            changeset: [
                                file        : filePath,
                                old_file    : oldPath,
                                file_type   : fileType,
                                change_type : changeType,
                                authors     : authors,
                                base_commit : baseCommit,
                                head_commit : headCommit,
                                diff_size   : diffSize,
                                diff        : [
                                    encoding: "base64",
                                    content : sh(
                                        script: "base64 -w 0 diff_current.txt",
                                        returnStdout: true
                                    ).trim()
                                ]
                            ]
                        ]

                        writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

                        /* === AI REVIEW === */
                        sh '''
                          echo "🚀 Sending diff to AI..."
                          HTTP_CODE=$(curl -s -o response.json -w "%{http_code}" \
                            -X POST "$WEBHOOK_URL" \
                            -H "Content-Type: application/json" \
                            -d @payload.json)

                          if [ "$HTTP_CODE" -ge 400 ]; then
                            echo "❌ AI Review failed"
                            cat response.json
                            exit 1
                          fi
                        '''

                        /* === AI TEST CASE === */
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
            echo "✅ AI Code Review completed"
        }
        always {
            archiveArtifacts artifacts: '*.txt,*.json,*.env', fingerprint: true
        }
    }
}
