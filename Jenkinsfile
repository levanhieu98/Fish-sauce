pipeline {
  agent any

  environment {
    GIT_CREDENTIAL = 'demo_github'
    WEBHOOK_URL = 'https://script.google.com/macros/s/AKfycbwYc5DYob3VyhMEBtmuyhATBbVE2E_iJlctyozHwkzn1tUivYldcyE1OUYns06ITtHs/exec'
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
          curl -X POST "$WEBHOOK_URL" \
          -H "Content-Type: application/json" \
          -d '{
            "repo": "Fish-sauce",
            "git_url": "https://github.com/levanhieu98/Fish-sauce.git",
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
