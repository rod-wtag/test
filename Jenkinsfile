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
                        def branchName = "release/${env.MAJOR}.${env.MINOR.toInteger() + 1}"

                        // Check if remote branch exists
                        def branchExists = sh(
                            script: "git ls-remote --heads origin ${branchName}",
                            returnStdout: true
                        ).trim()

                        if (branchExists == "") {
                            currentBuild.result = 'ABORTED'
                            error "Remote branch ${branchName} does not exist. Aborting."
                        }

                        echo "Remote branch ${branchName} exists. Proceeding."
                        sh "git fetch origin ${branchName}:${branchName}"
                        sh "git checkout ${branchName}"

                        // Try merging, allow failure
                        def mergeStatus = sh(
                            script: "git merge --no-commit --no-ff ${params.TAG} || true",
                            returnStatus: true
                        )

                        if (mergeStatus != 0) {
                            echo "Merge conflict detected."

                            def conflicted_files = sh(
                                script: "git diff --name-only --diff-filter=U",
                                returnStdout: true
                            ).trim()

                            echo "Conflicted files: ${conflicted_files}"

                            if (conflicted_files == "system/config/version.properties") {
                                echo "Only system/config/version.properties conflicted. Resolving by keeping branch version."

                                sh """
                                    git checkout --ours system/config/version.properties
                                    git add system/config/version.properties
                                    git commit -m 'Merged ${params.TAG} into ${branchName} - resolved version.properties conflict by keeping branch version.'
                                """
                            } else {
                                error("Conflict detected in files other than system/config/version.properties. Aborting.")
                            }
                        } else {
                            echo "Merge successful without conflicts."
                            sh "git commit -m 'Merged ${params.TAG} into ${branchName}'"
                        }
                    }
                }
            }
        }

    }
}