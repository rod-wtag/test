pipeline {
    agent any

    environment {
        GIT_CREDENTIALS_ID = 'github-creds'  // Your GitHub credentials ID
    }

    stages {
        stage('Checkout Branch') {
            steps {
                script {
                    echo "Checking out branch: ${env.GIT_BRANCH}"
                    // Checkout the branch that triggered the pipeline
                    sh """
                        git checkout ${env.GIT_BRANCH}
                    """
                }
            }
        }

        stage('Create File and Commit Changes') {
            steps {
                script {
                    echo "Creating a text file with Hello World"

                    // Create a new text file with Hello World content
                    sh """
                        echo 'Hello World' > hello.txt
                    """

                    // Stage, commit, and push the changes to the branch
                    withCredentials([usernamePassword(credentialsId: GIT_CREDENTIALS_ID, usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
                        sh """
                            git config user.name "rod-wtag"
                            git config user.email "roky.das@welldev.io"
                            git remote set-url origin https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/rod-wtag/git-flow-automation-jenkins.git

                            git add hello.txt
                            git commit -m "Added hello.txt with Hello World"
                            git push origin HEAD:${env.GIT_BRANCH}
                        """
                    }
                }
            }
        }
    }
}
