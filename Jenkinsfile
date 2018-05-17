pipeline {
 options {
    buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '30'))
  }
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building..'
            }
        }
    }
}