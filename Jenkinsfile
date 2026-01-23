pipeline {
  agent any

  environment {
    GIT_CREDENTIAL = 'demo_github'
    WEBHOOK_URL = 'https://script.google.com/a/macros/rivercrane.vn/s/AKfycbyLvd9wu5W7ASgI7M35VqJ6cmtK2L29em6t0zWXyjuCx5RZmLUnvfgm8zIpwHTqwg/exec'
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
          git diff HEAD~1 HEAD > diff.txt || true
        '''
      }
    }

    stage('Send to Gemini') {
      steps {
        sh '''
          curl -X POST "$WEBHOOK_URL" \
          -H "Content-Type: application/json" \
          -d '{
            "repo": "test",
            "branch": "main",
            "commit": "'"$(git rev-parse HEAD)"'",
            "author": "'"$(git log -1 --pretty=%an)"'",
            "diff": "'"$(sed 's/"/\\"/g' diff.txt)"'"
          }'
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
