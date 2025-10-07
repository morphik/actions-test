pipeline {
    agent any

    environment {
        REPO_URL = 'https://github.com/morphik/actions-test.git'
        GITHUB_CREDENTIALS = 'github-token'
        SOURCE_BRANCH = 'release1.3'
        TARGET_BRANCH = 'release1.4'
    }

    stages {
        stage('Debug Webhook Info') {
            steps {
                script {
                    echo "=== Webhook Debug Information ==="
                    echo "GIT_BRANCH: ${env.GIT_BRANCH ?: 'not set'}"
                    echo "GIT_COMMIT: ${env.GIT_COMMIT ?: 'not set'}"
                    echo "GIT_PREVIOUS_COMMIT: ${env.GIT_PREVIOUS_COMMIT ?: 'not set'}"
                    echo "GIT_URL: ${env.GIT_URL ?: 'not set'}"
                    echo "BUILD_CAUSE: ${currentBuild.buildCauses}"

                    // Print all environment variables for debugging
                    echo "=== All Environment Variables ==="
                    sh 'env | grep -i git || true'
                    echo "=============================="
                }
            }
        }

        stage('Checkout Repository') {
            steps {
                script {
                    deleteDir()

                    // Checkout with full history to find all commits
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${env.SOURCE_BRANCH}"]],
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [
                            [$class: 'CloneOption', depth: 0, noTags: false, shallow: false]
                        ],
                        submoduleCfg: [],
                        userRemoteConfigs: [[
                            credentialsId: env.GITHUB_CREDENTIALS,
                            url: env.REPO_URL
                        ]]
                    ])

                    echo "✅ Repository checked out successfully"
                }
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    // Get the latest commits from source branch
                    sh "git fetch origin ${env.SOURCE_BRANCH}"

                    // Check recent commits for PR merge indicators
                    def recentCommits = sh(
                        script: "git log -5 --oneline origin/${env.SOURCE_BRANCH}",
                        returnStdout: true
                    ).trim()

                    echo "=== Recent commits on ${env.SOURCE_BRANCH} ==="
                    echo recentCommits
                    echo "========================================="

                    // Look for PR merge patterns in recent commits
                    def prNumber = ""
                    def commitToSync = ""

                    // Check if latest commit is a PR merge (common patterns)
                    def latestCommitMsg = sh(
                        script: "git log -1 --pretty=%B origin/${env.SOURCE_BRANCH}",
                        returnStdout: true
                    ).trim()

                    echo "Latest commit message:"
                    echo latestCommitMsg

                    // Extract PR number from commit message using shell instead of regex
                    // Patterns: "Merge pull request #123" or "(#123)" or "PR #123"
                    def prNumberExtract = sh(
                        script: """
                            echo '${latestCommitMsg}' | grep -oE '#[0-9]+' | head -1 | sed 's/#//' || echo ''
                        """,
                        returnStdout: true
                    ).trim()

                    if (prNumberExtract) {
                        prNumber = prNumberExtract
                        echo "✅ Found PR number: #${prNumber}"
                    } else {
                        echo "⚠️ No PR number found in latest commit"
                    }

                    // Get the latest commit hash
                    commitToSync = sh(
                        script: "git rev-parse origin/${env.SOURCE_BRANCH}",
                        returnStdout: true
                    ).trim()

                    env.PR_NUMBER = prNumber ?: '0'
                    env.COMMIT_TO_SYNC = commitToSync
                    env.COMMIT_MESSAGE = latestCommitMsg

                    echo "=== Sync Information ==="
                    echo "PR Number: ${env.PR_NUMBER}"
                    echo "Commit: ${env.COMMIT_TO_SYNC}"
                    echo "======================="
                }
            }
        }

        stage('Sync Branches') {
            steps {
                script {
                    try {
                        sh '''
                            git config user.name "Jenkins Bot"
                            git config user.email "jenkins@yourcompany.com"
                        '''

                        sh 'git fetch origin'

                        // Checkout and update target branch
                        sh "git checkout ${env.TARGET_BRANCH}"
                        sh "git pull origin ${env.TARGET_BRANCH}"

                        echo "📋 Target branch ${env.TARGET_BRANCH} is at:"
                        sh "git log -1 --oneline"

                        // Checkout source branch
                        sh "git checkout ${env.SOURCE_BRANCH}"
                        sh "git pull origin ${env.SOURCE_BRANCH}"

                        def commitToSync = env.COMMIT_TO_SYNC

                        echo "🍒 Preparing to cherry-pick: ${commitToSync}"

                        // Switch to target branch for cherry-pick
                        sh "git checkout ${env.TARGET_BRANCH}"

                        // Check if it's a merge commit
                        def parentCount = sh(
                            script: "git rev-list --parents -n 1 ${commitToSync} | wc -w",
                            returnStdout: true
                        ).trim() as Integer
                        parentCount = parentCount - 1

                        if (parentCount > 1) {
                            echo "🔀 Merge commit detected (${parentCount} parents)"
                            echo "Using -m 1 to cherry-pick mainline"
                            sh "git cherry-pick -m 1 ${commitToSync}"
                        } else {
                            echo "📝 Regular commit - cherry-picking normally"
                            sh "git cherry-pick ${commitToSync}"
                        }

                        echo "📤 Pushing changes to ${env.TARGET_BRANCH}..."
                        withCredentials([string(credentialsId: env.GITHUB_CREDENTIALS, variable: 'GITHUB_TOKEN')]) {
                            sh """
                                git push https://x-access-token:${GITHUB_TOKEN}@github.com/morphik/actions-test.git ${env.TARGET_BRANCH}
                            """
                        }



                        env.SYNC_SUCCESS = 'true'
                        echo "✅ Successfully synced commit ${commitToSync} to ${env.TARGET_BRANCH}"

                    } catch (Exception e) {
                        env.SYNC_SUCCESS = 'false'
                        env.SYNC_ERROR = e.getMessage()
                        echo "❌ Sync failed: ${e.getMessage()}"

                        sh 'git cherry-pick --abort || true'

                        throw e
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                echo "✅ Pipeline completed successfully"

                if (env.PR_NUMBER != '0') {
                    try {
                        withCredentials([string(credentialsId: env.GITHUB_CREDENTIALS, variable: 'GITHUB_TOKEN')]) {
                            sh """
                                curl -s -X POST \\
                                    -H "Authorization: token \$GITHUB_TOKEN" \\
                                    -H "Content-Type: application/json" \\
                                    -d '{"body": "✅ **Jenkins Auto-sync Successful!**\\n\\nChanges from PR #${env.PR_NUMBER} have been synced to ${env.TARGET_BRANCH}.\\n\\n🔗 [Jenkins Build](${env.BUILD_URL})\\n📝 Commit: ${env.COMMIT_TO_SYNC}"}' \\
                                    "https://api.github.com/repos/morphik/actions-test/issues/${env.PR_NUMBER}/comments" \\
                                    || echo "Could not post comment to GitHub"
                            """
                        }
                    } catch (Exception e) {
                        echo "⚠️ Could not post comment: ${e.getMessage()}"
                    }
                }
            }
        }

        failure {
            script {
                echo "❌ Pipeline failed"

                if (env.PR_NUMBER != '0') {
                    try {
                        withCredentials([string(credentialsId: env.GITHUB_CREDENTIALS, variable: 'GITHUB_TOKEN')]) {
                            sh """
                                curl -s -X POST \\
                                    -H "Authorization: token \$GITHUB_TOKEN" \\
                                    -H "Content-Type: application/json" \\
                                    -d '{"title": "🚨 Jenkins sync failed: PR #${env.PR_NUMBER}", "body": "**Sync failed**\\n\\nSource: ${env.SOURCE_BRANCH}\\nTarget: ${env.TARGET_BRANCH}\\nError: ${env.SYNC_ERROR ?: 'Unknown'}\\n\\nBuild: ${env.BUILD_URL}", "labels": ["jenkins-sync-failed"]}' \\
                                    "https://api.github.com/repos/morphik/actions-test/issues" \\
                                    || echo "Could not create GitHub issue"
                            """
                        }
                    } catch (Exception e) {
                        echo "⚠️ Could not create issue: ${e.getMessage()}"
                    }
                }
            }
        }

        always {
            cleanWs()
        }
    }
}