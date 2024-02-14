pipeline {
  agent any
  stages {
    stage('Static Test') {
      when { branch 'develop' }
      steps {
        sh '''
          flake8 --format=pylint --exit-zero src > flake8.out
          bandit -r -o bandit.out -f custom --msg-template "{abspath}:{line}: [{test_id}] {msg}" --exit-zero src
        '''
        recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')]
        recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]
      }
    }
    stage('Deploy') {
      parallel {
        stage('Deploy - staging') {
          when { branch 'develop' }
          steps {
            sh '''
              sam build
              sam deploy --config-env staging
            '''
          }
        }
        stage('Deploy - production') {
          when { branch 'master' }
          steps {
            sh '''
              sam build
              sam deploy --config-env production
            '''
          }
        }
      }
    }
    stage('Rest Test') {
      parallel {
        stage('Rest Test - staging') {
          when { branch 'develop' }
          steps {
            sh '''
              export BASE_URL=$(sam list stack-outputs --config-env staging --output json | jq -r '.[] | select(.OutputKey=="BaseUrlApi") | .OutputValue')
              pytest-3 --junitxml=result-service.xml test/integration/todoApiTest.py
            '''
            junit 'result-service.xml'
          }
        }
        stage('Rest Test - production') {
          when { branch 'master' }
          steps {
            sh '''
              export BASE_URL=$(sam list stack-outputs --config-env production --output json | jq -r '.[] | select(.OutputKey=="BaseUrlApi") | .OutputValue')
              pytest-3 --junitxml=result-service.xml test/integration/todoApiTest_production.py
            '''
            junit 'result-service.xml'
          }
        }
      }
    }
    stage('Promote') {
      when { branch 'develop' }
      steps {
        deleteDir()
        withCredentials([gitUsernamePassword(credentialsId: 'github_jav956', gitToolName: 'git-tool')]) {
          sh """
            git clone -b develop 'https://github.com/jav956/todo-list-aws.git' .
            git checkout master
            git merge develop
            git push
          """
        }
      }
    }
  }
  post { 
    always { 
      deleteDir()
    }
  }
}