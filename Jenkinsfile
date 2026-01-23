pipeline {
    agent any

    environment {
        GIT_CREDENTIAL = 'demo_github'
        // Hãy đảm bảo URL này là bản Deploy mới nhất với quyền "Anyone"
        WEBHOOK_URL = 'https://script.google.com/macros/s/AKfycbz3RXMIYZNZvePrewV0RZjwghJK6KgJtZvd0Ut5xd2cT0ToouloCw_HFDbOLj3LvO2b9w/exec'
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
                    // Lấy các giá trị cần thiết bằng groovy để tránh lỗi nháy đơn/kép trong shell
                    def commitHash = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                    def authorName = sh(script: "git log -1 --pretty=%an", returnStdout: true).trim()
                    
                    sh """
                        # Mã hóa base64 và xóa các ký tự xuống dòng
                        DIFF_BASE64=\$(base64 diff.txt | tr -d '\\n')

                        # Tạo file payload.json để gửi thay vì truyền trực tiếp vào câu lệnh curl
                        cat <<EOF > payload.json
                        {
                            "repo": "Fish-sauce",
                            "git_url": "https://github.com/levanhieu98/Fish-sauce.git",
                            "branch": "main",
                            "commit": "${commitHash}",
                            "author": "${authorName}",
                            "diff_base64": "\$DIFF_BASE64"
                        }
                        EOF
                        echo "Gửi dữ liệu đến Webhook..."
                        # Sử dụng -d @filename để curl tự đọc file JSON chuẩn
                        RESPONSE=\$(curl -s -L -X POST "${env.WEBHOOK_URL}" \\
                            -H "Content-Type: application/json" \\
                            -d @payload.json)

                        echo "--- Webhook Response ---"
                        echo "\$RESPONSE"
                        echo "------------------------"
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