pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        GIT_CREDENTIALS_ID = 'github-creds'
        REPO_URL = 'github.com/rod-wtag/git-flow-automation-jenkins.git'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    // Set git config for commits
                    sh """
                        git config user.name "rod-wtag"
                        git config user.email "roky.das@welldev.io"
                    """
                    
                    // Store branch name for later use
                    env.BRANCH_NAME = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
                    echo "Working on branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Create Hello World File') {
            steps {
                script {
                    // Create a text file with Hello World
                    writeFile file: 'hello_world.txt', text: 'Hello World'
                    echo "Created hello_world.txt file"
                }
            }
        }

        stage('Commit and Push Changes') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
                        sh """
                            # Check if there are changes to commit
                            git add hello_world.txt
                            
                            # Commit only if there are changes
                            git diff --cached --quiet || git commit -m "Add hello_world.txt file"
                            
                            # Set the remote URL with credentials
                            git remote set-url origin https://${GIT_USERNAME}:${GIT_TOKEN}@${REPO_URL}
                            
                            # Push changes back to the same branch
                            git push origin HEAD:${env.BRANCH_NAME}
                            
                            echo "Successfully pushed changes to ${env.BRANCH_NAME}"
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}