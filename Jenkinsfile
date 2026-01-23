import groovy.json.JsonOutput

pipeline {
    agent any
    environment {
        GIT_CREDENTIAL = 'demo_github'
        // Đảm bảo URL này là bản mới nhất sau khi bạn nhấn "Triển khai"
        WEBHOOK_URL = 'https://script.google.com/macros/s/AKfycbxyGG9cqdOc6pQxW1xC0sUKZgweZiEU-aN78j3bYjM8OND_VMBsneHyEU2WnG_t_eh5zw/exec'
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
                        git_url: "https://github.com/levanhieu98/Fish-sauce.git",
                        branch: "main",
                        commit: commitHash,
                        author: authorName,
                        diff_base64: diffBase64
                    ]
                    
                    writeFile file: 'payload.json', text: JsonOutput.toJson(payload)

                    sh """
                        echo "--- Đang gửi Webhook ---"
                        # Bỏ cờ -f để đọc lỗi trả về
                        curl -s -L -X POST "${env.WEBHOOK_URL}?cache=clear" \
                             -H "Content-Type: application/json" \
                             -d @payload.json > response.json
                        
                        echo "Phản hồi từ Google:"
                        cat response.json
                        
                        # Nếu phản hồi chứa "error", cho build thất bại
                        if grep -q "error" response.json; then
                            exit 1
                        fi
                    """
                }
            }
        }
    }
}