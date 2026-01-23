pipeline {
  agent any

  environment {
    GIT_CREDENTIAL = 'demo_github'
    WEBHOOK_URL = 'https://script.google.com/macros/s/AKfycbxJxi5a67DWGDZtIp9URNHBuNkIIkCkMINyXkzchgD3gWwO8p2MvA9NYyYCDD9xI82ZKQ/exec'
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
          git fetch origin
          git diff HEAD~1 HEAD > diff.txt || git show HEAD > diff.txt
        '''
      }
    }

    stage('Send to Gemini') {
        steps {
            sh '''
            DIFF_BASE64=$(base64 diff.txt | tr -d '\\n')
            export DIFF_BASE64

            RESPONSE=$(curl -s -X POST "$WEBHOOK_URL" \
                -H "Content-Type: application/json" \
                -d '{
                "repo": "Fish-sauce",
                "git_url": "https://github.com/levanhieu98/Fish-sauce.git",
                "branch": "main",
                "commit": "'"$(git rev-parse HEAD)"'",
                "author": "'"$(git log -1 --pretty=%an)"'",
                "diff_base64": "'"$DIFF_BASE64"'"
                }')

            echo "Webhook response: $RESPONSE"
            '''
        }
    }
  }

  post {
    failure {
      echo 'Pipeline failed'
    }
  }
}
