pipeline {
    agent any

    environment {
        GIT_CREDENTIAL = 'demo_github'
        // Hãy đảm bảo URL này là bản Deploy mới nhất với quyền "Anyone"
        WEBHOOK_URL = 'https://script.google.com/macros/s/AKfycbyBaj2EU5rut27knfRHfSE8Fpbltc-SBt6-20IhXnvPnwaJyhb2Kjuv7wiVG9Kc027B6g/exec'
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
                    def commitHash = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                    def authorName = sh(script: "git log -1 --pretty=%an", returnStdout: true).trim()
                    
                    sh """
                        # Tạo DIFF_BASE64 và xóa các ký tự xuống dòng
                        DIFF_BASE64=\$(base64 diff.txt | tr -d '\\n')
                        
                        # Ghi JSON vào file payload.json bằng Groovy writeFile để an toàn 100%
                        # (Không dùng printf trong shell để tránh lỗi giới hạn ký tự dòng lệnh)
                    """
                    
                    // Cách an toàn nhất: Dùng writeFile của Jenkins để tạo file JSON
                    def jsonPayload = """{
                        "repo": "Fish-sauce",
                        "git_url": "https://github.com/levanhieu98/Fish-sauce.git",
                        "branch": "main",
                        "commit": "${commitHash}",
                        "author": "${authorName}",
                        "diff_base64": "${sh(script: "base64 diff.txt | tr -d '\\n'", returnStdout: true).trim()}"
                    }"""
                    
                    writeFile file: 'payload.json', text: jsonPayload

                    sh """
                        echo "Đang gửi dữ liệu đến Webhook..."
                        # Thêm cờ -i để xem header phản hồi nếu cần debug
                        curl -s -L -X POST "${env.WEBHOOK_URL}" \
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