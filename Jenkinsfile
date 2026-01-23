import groovy.json.JsonOutput

pipeline {
    agent any
    environment {
        GIT_CREDENTIAL = 'demo_github'
        // DÙNG URL MỚI NHẤT BẠN VỪA TẠO
        WEBHOOK_URL = 'https://script.google.com/macros/s/AKfycbwEdUX-KRMDozFInbwcBc9synUHHV_vq0VeHImj4ltekm_538L82q5Gv13w6h14yC1o/exec'
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
                        echo "--- Gửi thử nghiệm ---"
                        # Loại bỏ cờ -f và tham số URL để tránh 404
                        curl -s -L -X POST "${env.WEBHOOK_URL}" \
                             -H "Content-Type: application/json" \
                             --data-binary @payload.json > response.json
                        
                        echo "Kết quả từ Google:"
                        cat response.json
                    """
                }
            }
        }
    }
}