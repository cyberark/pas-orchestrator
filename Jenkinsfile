pipeline {
  agent {
    node {
      label 'ansible'
    }
  }
  stages {
    stage('ansible-lint validation') {
      steps {
        script {
          sh(script: "pip install ansible-lint --user", returnStdout: true)
          sh(script: "ansible-lint pas-orchestrator.yml", returnStdout: true)
        }
      }
    }
    stage('yamllint validation') {
      steps {
        script {
          sh(script: "pip install yamllint --user", returnStdout: true)
          sh(script: "yamllint pas-orchestrator.yml", returnStdout: true)
        }
      }
    }
  }
}
