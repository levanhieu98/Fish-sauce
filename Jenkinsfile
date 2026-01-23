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
                    // 1. Lấy thông tin commit
                    def commitHash = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                    def authorName = sh(script: "git log -1 --pretty=%an", returnStdout: true).trim()
                    
                    // 2. Tạo nội dung file JSON và ghi ra file bằng lệnh của Jenkins (writeJSON hoặc writeFile)
                    // Lưu ý: DIFF_BASE64 được tạo trực tiếp trong shell để xử lý file lớn an toàn
                    sh """
                        DIFF_BASE64=\$(base64 diff.txt | tr -d '\\n')
                        
                        # Tạo file payload bằng printf để tránh các lỗi ký tự đặc biệt
                        printf '{"repo":"Fish-sauce","git_url":"https://github.com/levanhieu98/Fish-sauce.git","branch":"main","commit":"%s","author":"%s","diff_base64":"%s"}' \
                        "$commitHash" "$authorName" "\$DIFF_BASE64" > payload.json
                    """

                    // 3. Gửi Webhook
                    sh """
                        echo "Đang gửi dữ liệu đến Webhook..."
                        if [ -f payload.json ]; then
                            RESPONSE=\$(curl -s -L -X POST "${env.WEBHOOK_URL}" \
                                -H "Content-Type: application/json" \
                                -d @payload.json)
                            
                            echo "--- Webhook Response ---"
                            echo "\$RESPONSE"
                            echo "------------------------"
                        else
                            echo "LỖI: Không tìm thấy file payload.json"
                            exit 1
                        fi
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