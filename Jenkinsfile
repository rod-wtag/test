pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        GIT_CREDENTIALS_ID = 'github-creds'
        REPO_URL = 'github.com/rod-wtag/test.git'
        USER_NAME = 'rod-wtag'
        USER_EMAIL = 'roky.das@welldev.io'
    }

    stages {
        stage('Checkout and Refresh Branch') {
            steps {
                checkout scm
                script {
                    // Set git config for commits
                    sh """
                        git config user.name ${env.USER_NAME}
                        git config user.email ${env.USER_EMAIL}
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

        stage('Check Branch') {
            steps {
                script {
                    echo "Current branch: ${env.CURRENT_BRANCH}"
                    if (!env.CURRENT_BRANCH.contains('release/')) {
                        echo "Not detected release branch. Stopping pipeline execution."
                        currentBuild.result = 'ABORTED'
                        error "Pipeline stopped because it's not running on release branch"
                    } else {
                        echo "It's a release branch, continuing with pipeline execution"
                    }
                }
            }
        }

        stage('Check Commit Message') {
            steps {
                script {
                    // Get latest commit message
                    def commitMessage = sh(
                        script: "git log -1 --pretty=%B",
                        returnStdout: true
                    ).trim()

                    echo "Latest Commit Message: '${commitMessage}'"

                    if (commitMessage.toLowerCase().contains('bump version')) {
                        echo "Commit contains 'bump version'. Aborting pipeline."
                        // Use 'error' to fail and abort the pipeline

                        currentBuild.result = 'ABORTED'
                        error("Pipeline aborted due to 'bump version' commit.")
                    }
                }
            }
        }

        stage('Get and update bump version') {
            steps {
                script {

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

                    // Working with fresh branch from previous stage
                    def versionFilePath = 'system/config/version.properties'
                    env.VERSION_FILE_PATH = versionFilePath
                    def versionFileContent = readFile(versionFilePath).trim()

                    // Extract the version line
                    def versionLine = versionFileContent.readLines().find { it.startsWith('version:') }
                    if (!versionLine) {
                        error "version: line not found in properties file"
                    }

                    // Parse version numbers using split
                    def versionStr = versionLine.split(':')[1].trim()
                    def (major, minor, patch) = versionStr.split('\\.').collect { it.toInteger() }
                    
                    // Bump the patch
                    patch += 1
                    def newVersion = "${major}.${minor}.${patch}"
                    env.MAJOR = major
                    env.MINOR = minor
                    env.VERSION = newVersion
                    env.TAG_NAME = "r${major}.${minor}.${patch}"
                    echo "Bump version: ${env.VERSION}"

                    // Replace the version line
                    def updatedContent = "version: ${newVersion}"
                    writeFile(file: versionFilePath, text: updatedContent)

                    // Create a text file with Hello World
                    // writeFile file: 'hello_world1.txt', text: 'Hello World'
                    // echo "Created hello_world.txt file"
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
                            git add ${env.VERSION_FILE_PATH}
                            
                            # Commit
                            git commit -m "bump version ${env.VERSION}" || echo "No changes to commit"
                            
                            # Push changes
                            git push origin HEAD:${env.CURRENT_BRANCH}
                            
                            echo "Successfully pushed changes to ${env.CURRENT_BRANCH}"
                        """
                    }
                }
            }
        }

        stage('Tag & Push') {
            steps {
                echo "Creating Tag & Push for branch: ${env.CURRENT_BRANCH}"
                withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
                    sh """
                        git tag -d ${env.TAG_NAME} || true
                        git tag -f ${env.TAG_NAME}
                        git push https://${GIT_USERNAME}:${GIT_TOKEN}@${REPO_URL} ${env.TAG_NAME}
                    """
                }
            }
        }

        stage('Merge Tag to upper branch') {
            steps {
                echo "Running Merge Tag for branch: ${env.CURRENT_BRANCH}"
                withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
                    script {
                        def branchName = "release/${env.MAJOR}.${env.MINOR}"

                        sh """
                            set -e  # Fail on any error

                            git fetch --all

                            if git show-ref --verify --quiet refs/heads/${branchName}; then
                                echo "Branch '${branchName}' exists. Proceeding to merge."

                                git branch -D ${branchName}-local || true
                                git checkout origin/${branchName} -b ${branchName}-local
                            else
                                echo "Branch '${branchName}' does not exist. Creating merge branch instead."

                                git checkout origin/main -B main-local
                                git checkout -b merge/${env.TAG_NAME}
                            fi

                            # Try merging the tag
                            set +e  # Allow merge conflicts temporarily
                            git merge ${env.TAG_NAME} -m 'Merge tag ${env.TAG_NAME}'

                            merge_exit_code=$?
                            set -e  # Back to strict mode

                            if [ $merge_exit_code -ne 0 ]; then
                                echo "Merge conflicts detected. Checking conflicted files..."

                                conflicted_files=$(git diff --name-only --diff-filter=U)

                                # Check if conflicts only happen in system/config/version.properties
                                if echo "$conflicted_files" | grep -qv "^system/config/version.properties$"; then
                                    echo "Conflicts found in files other than version.properties. Aborting merge."
                                    git merge --abort
                                    exit 1
                                else
                                    echo "Only version.properties is conflicted. Resolving by keeping existing version."

                                    # Checkout our version of the file
                                    git checkout --ours system/config/version.properties
                                    git add system/config/version.properties

                                    # Continue merge
                                    git commit -m "Resolve version.properties conflict by keeping existing content."
                                fi
                            fi

                            # Push the merged branch
                            CURRENT_BRANCH=$(git branch --show-current)
                            git push https://${GIT_USERNAME}:${GIT_TOKEN}@${REPO_URL} HEAD:${CURRENT_BRANCH}
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