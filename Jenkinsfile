import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        GIT_CREDENTIAL = 'demo_github'
        BASE_BRANCH = 'origin/main'
        WEBHOOK_URL = 'https://script.google.com/macros/s/AKfycbwKJ4Xh0v02OdTUbS96Ie-cvZno1INGrN8Ex7KtLEWrVm9LfjH1x1F9MO-lvHkeBIrQ/exec'
        PROJECT_NAME = 'Fish-sauce'
    }

    stages {

        stage('Checkout') {
            steps {
                git(
                    url: 'https://github.com/levanhieu98/Fish-sauce.git',
                    branch: 'main',
                    credentialsId: env.GIT_CREDENTIAL
                )
                sh 'git fetch origin main'
            }
        }

        stage('Collect Diff') {
            steps {
                sh '''
                  echo "--- Collecting git diff ---"

                  # Nếu có commit trước → diff bình thường
                  if git rev-parse HEAD~1 >/dev/null 2>&1; then
                    git diff HEAD~1 HEAD > diff.txt
                  else
                    # Commit đầu tiên
                    git show HEAD > diff.txt
                  fi

                  echo "--- Diff preview ---"
                  head -200 diff.txt
                '''
            }
        }

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
                        error "❌ Diff quá nhỏ hoặc rỗng – không gửi AI review"
                    }

                    // Encode base64 để tránh lỗi ký tự
                    def diffBase64 = sh(
                        script: "base64 diff.txt | tr -d '\\n'",
                        returnStdout: true
                    ).trim()

                    def payload = [
                        project      : PROJECT_NAME,
                        commit       : commitHash,
                        author       : authorName,
                        diff_base64  : diffBase64,
                        diff_size    : diffSize
                    ]

                    writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

                    sh '''
                        echo "--- Sending payload to Gemini ---"
                        curl -s -L -X POST "$WEBHOOK_URL" \
                          -H "Content-Type: application/json" \
                          -d @payload.json > response.json

                        echo "--- Response ---"
                        cat response.json
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ AI Code Review pipeline completed"
        }
        failure {
            echo "❌ AI Code Review pipeline failed"
        }
    }
}



// ========================================
// import groovy.json.JsonOutput

// pipeline {
//     agent any
//     environment {
//         GIT_CREDENTIAL = 'demo_github'
//         // DÙNG URL MỚI NHẤT BẠN VỪA TẠO
//         WEBHOOK_URL = 'https://script.google.com/macros/s/AKfycbwKJ4Xh0v02OdTUbS96Ie-cvZno1INGrN8Ex7KtLEWrVm9LfjH1x1F9MO-lvHkeBIrQ/exec'
//     }
//     stages {
//         stage('Checkout') {
//             steps {
//                 git(
//                     url: 'https://github.com/levanhieu98/Fish-sauce.git', 
//                     branch: 'main', 
//                     credentialsId: env.GIT_CREDENTIAL
//                 )
//             }
//         }
//         stage('Collect Diff') {
//             steps {
//                 sh 'git fetch origin main && (git diff HEAD~1 HEAD > diff.txt || git show HEAD > diff.txt)'
//             }
//         }
//         stage('Send to Gemini') {
//             steps {
//                 script {
//                     def commitHash = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
//                     def authorName = sh(script: "git log -1 --pretty=%an", returnStdout: true).trim()
//                     def diffBase64 = sh(script: "base64 diff.txt | tr -d '\\n'", returnStdout: true).trim()
                    
//                     def payload = [
//                         repo: "Fish-sauce",
//                         author: authorName,
//                         diff_base64: diffBase64
//                     ]
                    
//                     writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

//                    sh """
//                         echo "--- Đang gửi yêu cầu Review ---"
//                         curl -s -L -X POST "${env.WEBHOOK_URL}" \
//                             -H "Content-Type: application/json" \
//                             -d @payload.json > response.json
                        
//                         if grep -q "success" response.json; then
//                             echo "✅ Đã gửi dữ liệu thành công! Kiểm tra Google Sheet và Google Chat nhé."
//                         else
//                             echo "⚠️ Có phản hồi nhưng có thể bị Redirect. Hãy kiểm tra Google Sheet."
//                         fi
//                     """
//                 }
//             }
//         }
//     }
// }