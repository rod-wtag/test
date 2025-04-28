pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        GIT_CREDENTIALS_ID = 'github-creds'
        REPO_URL = 'github.com/rod-wtag/test.git'  // Updated to match your actual repo
    }

    options {
        // Clean workspace before each build
        skipDefaultCheckout(true)
    }

    stages {
        stage('Clean Workspace') {
            steps {
                // Clean workspace completely
                cleanWs()
                
                // Fresh checkout
                // checkout([
                //     $class: 'GitSCM',
                //     branches: [[name: "${env.GIT_BRANCH}"]],
                //     doGenerateSubmoduleConfigurations: false,
                //     extensions: [[$class: 'CleanBeforeCheckout']],
                //     userRemoteConfigs: [[
                //         credentialsId: "${GIT_CREDENTIALS_ID}",
                //         url: "https://${REPO_URL}"
                //     ]]
                // ])
            }
        }

        stage('Configure Git') {
            steps {
                script {
                    // Set git config for commits
                    sh """
                        git config user.name "rod-wtag"
                        git config user.email "roky.das@welldev.io"
                    """
                    
                    // Store correct branch name from GIT_BRANCH environment variable
                    env.CURRENT_BRANCH = env.GIT_BRANCH.replaceAll('origin/', '')
                    echo "Working on branch: ${env.CURRENT_BRANCH}"
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
                            
                            # Add the file
                            git add hello_world.txt
                            
                            # Commit
                            git commit -m "Add hello_world.txt file"
                            
                            # Force push to overwrite any history
                            git push --force origin HEAD:${env.CURRENT_BRANCH}
                            
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
        always {
            // Clean up workspace after build
            cleanWs()
        }
    }
}