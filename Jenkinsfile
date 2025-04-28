pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        GIT_CREDENTIALS_ID = 'github-creds'
        REPO_URL = 'github.com/rod-wtag/test.git'
    }

    stages {
        stage('Checkout and Refresh Branch') {
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
                    
                    // Delete local branch if it exists and create fresh from origin
                    sh """
                        # Fetch all from remote
                        git fetch origin
                        
                        # Check if we're in detached HEAD state and save current commit
                        CURRENT_SHA=\$(git rev-parse HEAD)
                        
                        # Delete local branch if it exists (ignoring errors if it doesn't)
                        git checkout -f \$CURRENT_SHA
                        git branch -D ${env.CURRENT_BRANCH} || true
                        
                        # Create fresh branch from origin
                        git checkout -B ${env.CURRENT_BRANCH} origin/${env.CURRENT_BRANCH}
                        
                        # Verify we're on a clean branch
                        git status
                    """
                }
            }
        }

        stage('Create Hello World File') {
            steps {
                script {
                    // Create a text file with Hello World
                    writeFile file: 'hello_world1.txt', text: 'Hello World'
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
                            
                            # Add the file
                            git add hello_world1.txt
                            
                            # Commit
                            git commit -m "Add hello_world.txt file"
                            
                            # Push changes
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