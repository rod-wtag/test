pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        GIT_CREDENTIALS_ID = 'github-creds'
    }

    stages {
        stage('adding username and email') {
            steps {
                script {
                    sh """
                        git config user.name "rod-wtag"
                        git config user.email "roky.das@welldev.io"
                    """
                }
            }
        }
        

        stage('Check Branch') {
            steps {
                script {
                    echo "Current branch: ${env.GIT_BRANCH}"
                    if (!env.GIT_BRANCH.contains('release/')) {
                        echo "Not detected release branch. Stopping pipeline execution."
                        currentBuild.result = 'ABORTED'
                        error "Pipeline stopped because it's not running on release branch"
                    } else {
                        echo "It's a release branch, continuing with pipeline execution"
                    }
                }
            }
        }

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

        stage('Get and update bump version') {
            steps {
                script {
                    // Ensure we are on the correct branch and pull with rebase to avoid conflicts
                    sh """
                        git checkout ${env.GIT_BRANCH}
                        git pull --rebase origin ${env.GIT_BRANCH}
                    """

                    def versionFilePath = 'system/config/version.properties'
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

                    // Commit changes and push
                    withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
                        sh """
                            git config user.name "rod-wtag"
                            git config user.email "roky.das@welldev.io"
                            git remote set-url origin https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/rod-wtag/git-flow-automation-jenkins.git

                            git add ${versionFilePath}
                            git commit -m "bump version ${env.VERSION}" || echo "No changes to commit"

                            # Push changes
                            git push origin HEAD:${env.GIT_BRANCH}
                        """
                    }
                }
            }
        }


        stage('Tag & Push') {
            steps {
                echo "Creating Tag & Push for branch: ${env.GIT_BRANCH}"
                withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
                    sh '''
                        git config user.name "rod-wtag"
                        git config user.email "roky.das@welldev.io"

                        git tag -d ${env.TAG_NAME} || true
                        git tag -f ${env.TAG_NAME}
                        git push https://$GIT_USERNAME:$GIT_TOKEN@github.com/rod-wtag/git-flow-automation-jenkins.git $env.TAG_NAME
                    '''
                }
            }
        }


        stage('Merge Tag to upper branch') {
            steps {
                echo "Running Merge Tag for branch: ${env.GIT_BRANCH}"
                withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
                    script {
                        def branchName = "release/${env.MAJOR}.${env.MINOR}"

                        sh '''
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
                            git merge ${env.TAG_NAME} -m "Merge tag ${env.TAG_NAME}"

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
                            git push https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/rod-wtag/git-flow-automation-jenkins.git HEAD:${CURRENT_BRANCH}
                        '''
                    }
                }
            }
        }
    }
}