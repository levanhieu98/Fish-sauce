import groovy.json.JsonOutput 

pipeline {
    agent any

    environment {
        GIT_CREDENTIAL = 'demo_github'
        // CẬP NHẬT URL MỚI NHẤT TẠI ĐÂY
        WEBHOOK_URL = 'https://script.google.com/macros/s/AKfycbw0YW2Bmxo8gGHJPef-x-5wCv5UfKAtPexf8U2K_EdPWtcGmmKQ51MtXXCH9cUubeL6pw/exec'

    }

    stages {
        stage('Checkout') {
            steps {
                git(
                    url: 'https://github.com/levanhieu98/Fish-sauce.git',
                    branch: 'main',
                    credentialsId: env.GIT_CREDENTIAL
                )
            }
        }

        stage('Collect Diff') {
            steps {
                sh '''
                    git fetch origin main
                    git diff HEAD~1 HEAD > diff.txt || git show HEAD > diff.txt
                '''
            }
        }

        stage('Send to Gemini') {
            steps {
                script {
                    def commitHash = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                    def authorName = sh(script: "git log -1 --pretty=%an", returnStdout: true).trim()
                    def diffBase64 = sh(script: "base64 diff.txt | tr -d '\\n'", returnStdout: true).trim()
                    
                    // Tạo payload JSON an toàn bằng Groovy
                    def payload = [
                        repo: "Fish-sauce",
                        git_url: "https://github.com/levanhieu98/Fish-sauce.git",
                        branch: "main",
                        commit: commitHash,
                        author: authorName,
                        diff_base64: diffBase64
                    ]
                    
                    // Chuyển đối tượng sang chuỗi JSON và ghi file
                    def jsonString = JsonOutput.toJson(payload)
                    writeFile file: 'payload.json', text: jsonString

                    sh """
                        echo "Đang gửi dữ liệu đến Webhook..."
                        # Sử dụng -f để Jenkins thất bại nếu Google trả về lỗi (404/500)
                        curl -f -s -L -X POST "${env.WEBHOOK_URL}" \
                             -H "Content-Type: application/json" \
                             -d @payload.json
                    """
                }
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed. Vui lòng kiểm tra lại cấu hình Webhook hoặc quyền truy cập của Google Script.'
        }
    }
}