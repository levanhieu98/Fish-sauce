import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        WEBHOOK_URL  = 'https://script.google.com/macros/s/AKfycby_GMrTUo2vCpRv3mzRfnW3CUsYhbQTf9p5jkRZqrG2VZpdodYsPWmn9r4NLZyW6I51/exec'
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
    //     stage('Collect Diff') {
    //       steps {
    //           sh '''
    //             echo "Collecting PR diff..."

    //             BASE_REF="base/${CHANGE_TARGET}"

    //             # Fetch base branch vào local ref riêng
    //             git fetch origin ${CHANGE_TARGET}:refs/remotes/${BASE_REF}

    //             # Diff giống GitHub PR
    //             git diff ${BASE_REF}...HEAD > diff.txt

    //             if [ ! -s diff.txt ]; then
    //               echo "⏭️ No code changes in PR – skip AI review"
    //               exit 0
    //             fi

    //             echo "---- Diff preview ----"
    //             head -200 diff.txt
    //           '''
    //       }
    //   }

    stage('Collect Diff Per File') {
        steps {
            script {
            sh '''
                git fetch origin ${CHANGE_TARGET}
                git diff origin/${CHANGE_TARGET}...HEAD --name-only > files_raw.txt
            '''

            def allowExt = ['.php', '.js', '.ts', '.vue', '.go', '.py']
            def denyFiles = [
                'package-lock.json',
                'yarn.lock',
                'pnpm-lock.yaml',
                'composer.lock',
                'go.sum',
                'poetry.lock'
            ]

            def files = readFile('files_raw.txt')
                .split('\n')
                .collect { it.trim() }
                .findAll { f ->
                f &&
                allowExt.any { f.endsWith(it) } &&
                !denyFiles.contains(f)
                }

            if (files.isEmpty()) {
                echo "⏭️ No reviewable files"
                return
            }

            files.each { f ->
                sh """
                git diff origin/${CHANGE_TARGET}...HEAD -- ${f} > diff_${f.replaceAll('/', '_')}.txt
                """
            }
            }
        }
    }

        /* =========================
           SEND TO GEMINI
        ========================== */
        //   stage('Send to Gemini') {
        //       steps {
        //           script {
        //               // 1. Tính size diff
        //               def diffSize = sh(
        //                   script: "wc -c diff.txt | awk '{print \$1}'",
        //                   returnStdout: true
        //               ).trim()

        //               if (diffSize.toInteger() < 50) {
        //                   echo "⏭️ Diff too small – skip Gemini review"
        //                   return
        //               }

        //               // 2. Build payload
        //               def payload = [
        //                   project      : env.JOB_NAME,
        //                   repo         : env.JOB_NAME,
        //                   commit       : sh(script: 'git rev-parse HEAD', returnStdout: true).trim(),
        //                   author       : sh(script: 'git log -1 --pretty=%an', returnStdout: true).trim(),
        //                   diff_base64  : sh(script: "base64 diff.txt | tr -d '\\n'", returnStdout: true).trim(),
        //                   diff_size    : diffSize.toInteger(),
        //                   pr_id        : env.CHANGE_ID ?: "",
        //                   pr_branch    : env.CHANGE_BRANCH ?: "",
        //                   base_branch  : env.CHANGE_TARGET ?: "",
        //                   build_number : env.BUILD_NUMBER,
        //                   build_url    : env.BUILD_URL
        //               ]

        //               // 3. Ghi file JSON (pretty để debug)
        //               writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

        //               // 4. Debug + gửi webhook
        //               sh '''
        //                   echo "--- Sending payload to Gemini ---"
        //                   curl -s -L -X POST "$WEBHOOK_URL" \
        //                     -H "Content-Type: application/json" \
        //                     -d @payload.json > response.json

        //                   echo "--- Response ---"
        //               '''
        //           }
        //       }
        //   }

        stage('Send to Gemini') {
            steps {
                script {

                echo "🔍 Collect changed files in PR..."

                // 1️⃣ Lấy danh sách file thay đổi trong PR
                sh """
                    git fetch origin ${env.CHANGE_TARGET}
                    git diff origin/${env.CHANGE_TARGET}...HEAD --name-only > files_changed.txt
                """

                def files = readFile('files_changed.txt')
                    .split('\n')
                    .collect { it.trim() }
                    .findAll { it }

                echo "📂 Changed files:"
                files.each { echo " - ${it}" }

                // 2️⃣ Auto skip HTML + lockfile
                def reviewFiles = files.findAll { f ->
                    !(f ==~ /(\.html$|package-lock\.json$|yarn\.lock$|pnpm-lock\.yaml$|composer\.lock$|go\.sum$|poetry\.lock$)/)
                }

                if (reviewFiles.isEmpty()) {
                    echo "⏭️ Only HTML / lockfiles detected → skip Gemini review"
                    return
                }

                echo "🧠 Files sent to Gemini:"
                reviewFiles.each { echo " ✅ ${it}" }

                // 3️⃣ Tạo diff CHỈ cho file cần review
                sh """
                    > diff.txt
                    for f in ${reviewFiles.join(' ')}; do
                    echo "===== FILE: \$f =====" >> diff.txt
                    git diff origin/${env.CHANGE_TARGET}...HEAD -- "\$f" >> diff.txt
                    echo "" >> diff.txt
                    done
                """

                // 4️⃣ Check diff size
                def diffSize = sh(
                    script: "wc -c diff.txt | awk '{print \$1}'",
                    returnStdout: true
                ).trim().toInteger()

                if (diffSize < 50) {
                    echo "⏭️ Diff too small after filtering → skip Gemini"
                    return
                }

                // 5️⃣ Build payload
                def payload = [
                    project      : env.JOB_NAME,
                    repo         : env.JOB_NAME,
                    commit       : sh(script: 'git rev-parse HEAD', returnStdout: true).trim(),
                    author       : sh(script: 'git log -1 --pretty=%an', returnStdout: true).trim(),
                    diff_base64  : sh(script: "base64 diff.txt | tr -d '\\n'", returnStdout: true).trim(),
                    diff_size    : diffSize,
                    pr_id        : env.CHANGE_ID ?: "",
                    pr_branch    : env.CHANGE_BRANCH ?: "",
                    base_branch  : env.CHANGE_TARGET ?: "",
                    files        : reviewFiles,
                    build_number : env.BUILD_NUMBER,
                    build_url    : env.BUILD_URL
                ]

                // 6️⃣ Ghi JSON
                writeFile file: 'payload.json',
                    text: groovy.json.JsonOutput.prettyPrint(
                    groovy.json.JsonOutput.toJson(payload)
                    )

                // 7️⃣ Gửi Gemini
                sh '''
                    echo "===== PAYLOAD ====="
                    cat payload.json
                    echo "==================="

                    curl -s -L -X POST "$WEBHOOK_URL" \
                    -H "Content-Type: application/json" \
                    --fail \
                    --data @payload.json
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
