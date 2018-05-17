pipeline {
 options {
    buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))
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