pipeline {
  agent any
  stages {
    stage('Setup Docker') {
      steps {
        script {
          // Add this step to wait for the dind service to be ready
          // The `until` loop keeps trying `docker info` until it succeeds
          sh 'until docker info; do echo Waiting for dind...; sleep 1; done'
        }
      }
    }
    stage('Build Docker Image') {
      steps {
        script {
          // Now, perform your docker commands
          sh 'docker build -t 28197b424a83bd0b563a3e4c98f4fac8aa681367 -f ./Dockerfile.witness .'
        }
      }
    }
  }
}
