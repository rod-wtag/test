pipeline {
    agent any

    triggers {
        githubPush()
    }

    stages {
        stage('Check Branch') {
            steps {
                echo 'hello release/21.15'
            }
        }
    }
}
