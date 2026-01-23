pipeline {
    agent any

    environment {
        GIT_CREDENTIAL = 'demo_github'
        // Hãy đảm bảo URL này là bản Deploy mới nhất với quyền "Anyone"
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
                    # Lấy diff, nếu commit đầu tiên thì dùng git show
                    git diff HEAD~1 HEAD > diff.txt || git show HEAD > diff.txt
                '''
            }
        }

        stage('Send to Gemini') {
            steps {
                script {
                    // Lấy thông tin commit
                    def commitHash = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                    def authorName = sh(script: "git log -1 --pretty=%an", returnStdout: true).trim()
                    def diffContentBase64 = sh(script: "base64 diff.txt | tr -d '\\n'", returnStdout: true).trim()
                    
                    // Tạo đối tượng JSON một cách an toàn
                    def payload = [
                        repo: "Fish-sauce",
                        git_url: "https://github.com/levanhieu98/Fish-sauce.git",
                        branch: "main",
                        commit: commitHash,
                        author: authorName,
                        diff_base64: diffContentBase64
                    ]
                    
                    // Viết file bằng hàm writeJSON (nếu có plugin) hoặc writeFile đơn giản
                    // Lưu ý: Sử dụng thư viện Groovy để tạo JSON chuẩn
                    import groovy.json.JsonOutput
                    def jsonString = JsonOutput.toJson(payload)
                    writeFile file: 'payload.json', text: jsonString

                    sh """
                        echo "Đang gửi dữ liệu đến Webhook..."
                        # Dùng -f để curl báo lỗi nếu Google trả về 404/500
                        curl -f -s -L -X POST "${env.WEBHOOK_URL}" \
                            -H "Content-Type: application/json" \
                            -d @payload.json || echo "Lỗi: Không thể kết nối Webhook. Kiểm tra quyền 'Anyone' của Google Script."
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