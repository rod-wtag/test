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
        stage('Checkout and Configure Git') {
            steps {
                checkout scm
                script {
                    // Set git config for commits
                    sh """
                        git config user.name "rod-wtag"
                        git config user.email "roky.das@welldev.io"
                    """
                    
                    // Store correct branch name from GIT_BRANCH environment variable
                    env.CURRENT_BRANCH = env.GIT_BRANCH.replaceAll('origin/', '')
                    echo "Working on branch: ${env.CURRENT_BRANCH}"
                    
                    // Explicitly checkout the branch to avoid detached HEAD
                    sh """
                        git checkout ${env.CURRENT_BRANCH}
                        git pull origin ${env.CURRENT_BRANCH} --rebase
                    """
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
                            # Set the remote URL with credentials
                            git remote set-url origin https://${GIT_USERNAME}:${GIT_TOKEN}@${REPO_URL}
                            
                            # Check if there are changes to commit
                            git add hello_world.txt
                            
                            # Commit only if there are changes
                            git diff --cached --quiet || git commit -m "Add hello_world.txt file"
                            
                            # Pull any new changes with rebase to avoid merge commits
                            git pull --rebase origin ${env.CURRENT_BRANCH}
                            
                            # Push changes back to the same branch
                            git push origin HEAD:${env.CURRENT_BRANCH}
                            
                            echo "Successfully pushed changes to ${env.CURRENT_BRANCH}"
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