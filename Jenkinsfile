import groovy.json.JsonOutput

pipeline {
    agent any
    environment {
        GIT_CREDENTIAL = 'demo_github'
        // DÙNG URL MỚI NHẤT BẠN VỪA TẠO
        WEBHOOK_URL = 'https://script.google.com/macros/s/AKfycbzdGapHHakI0EYk8ZRVcDca7oc1edCJOoIHGUYYB7soiDhQpye5Nr-fpmNhWa0udRSg/exec'
    }
    stages {
        stage('Checkout') {
            steps {
                git(url: 'https://github.com/levanhieu98/Fish-sauce.git', branch: 'main', credentialsId: env.GIT_CREDENTIAL)
            }
        }
        stage('Collect Diff') {
            steps {
                sh 'git fetch origin main && (git diff HEAD~1 HEAD > diff.txt || git show HEAD > diff.txt)'
            }
        }
        stage('Send to Gemini') {
            steps {
                script {
                    def commitHash = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                    def authorName = sh(script: "git log -1 --pretty=%an", returnStdout: true).trim()
                    def diffBase64 = sh(script: "base64 diff.txt | tr -d '\\n'", returnStdout: true).trim()
                    
                    def payload = [
                        repo: "Fish-sauce",
                        author: authorName,
                        diff_base64: diffBase64
                    ]
                    
                    writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

                   sh """
                        echo "--- Đang gửi yêu cầu Review ---"
                        curl -s -L -X POST "${env.WEBHOOK_URL}" \
                            -H "Content-Type: application/json" \
                            -d @payload.json > response.json
                        
                        if grep -q "success" response.json; then
                            echo "✅ Đã gửi dữ liệu thành công! Kiểm tra Google Sheet và Google Chat nhé."
                        else
                            echo "⚠️ Có phản hồi nhưng có thể bị Redirect. Hãy kiểm tra Google Sheet."
                            cat response.json
                        fi
                    """
                }
            }
        }
    }
}