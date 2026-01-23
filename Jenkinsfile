pipeline {
  agent any

  environment {
    GIT_CREDENTIAL = 'demo_github'
    WEBHOOK_URL = 'https://script.google.com/macros/s/AKfycbx6x1kIAOhFYUcdOH6eIzfCnxfKYcv8xsOMtScRi4-OHypue7f_2LJ-TmAuvB66EV5zBA/exec'
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
        DIFF_BASE64=$(base64 -w 0 diff.txt)

        curl -X POST "$WEBHOOK_URL" \
            -H "Content-Type: application/json" \
            -d '{
            "repo": "Fish-sauce",
            "git_url": "https://github.com/levanhieu98/Fish-sauce.git",
            "branch": "main",
            "commit": "'"$(git rev-parse HEAD)"'",
            "author": "'"$(git log -1 --pretty=%an)"'",
            "diff_base64": "'"$DIFF_BASE64"'"
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
