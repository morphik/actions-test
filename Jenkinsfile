pipeline {
    agent any

    // Trigger on webhook from GitHub
    triggers {
        githubPullRequests {
            gitHubRepositoryName('morphik/actions-test')
            cron('')
            spec('H/5 * * * *')
            preStatus(false)
            events {
                pullRequestMerged()
            }
        }
    }

    environment {
        // Set your repository URL and credentials
        REPO_URL = 'https://github.com/morphik/actions-test.git'
        GITHUB_CREDENTIALS = 'github-token' // Jenkins credential ID
        SOURCE_BRANCH = 'release1.3'
        TARGET_BRANCH = 'release1.4'
    }

    stages {
        stage('Checkout Repository') {
            steps {
                script {
                    // Clean workspace
                    deleteDir()

                    // Clone repository with all branches
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: '*/*']],
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [
                            [$class: 'CloneOption', depth: 0, noTags: false, reference: '', shallow: false],
                            [$class: 'LocalBranch', localBranch: env.SOURCE_BRANCH]
                        ],
                        submoduleCfg: [],
                        userRemoteConfigs: [[
                            credentialsId: env.GITHUB_CREDENTIALS,
                            url: env.REPO_URL
                        ]]
                    ])
                }
            }
        }

        stage('Detect PR Info') {
            steps {
                script {
                    // Get PR information from environment variables set by webhook
                    env.PR_NUMBER = env.ghprbPullId ?: '0'
                    env.PR_TITLE = env.ghprbPullTitle ?: 'Unknown'
                    env.PR_AUTHOR = env.ghprbPullAuthorLogin ?: 'unknown'

                    echo "Processing PR #${env.PR_NUMBER}: ${env.PR_TITLE}"
                    echo "Author: ${env.PR_AUTHOR}"
                    echo "Source Branch: ${env.SOURCE_BRANCH}"
                    echo "Target Branch: ${env.TARGET_BRANCH}"
                }
            }
        }

        stage('Sync Branches') {
            steps {
                script {
                    try {
                        // Configure git
                        sh '''
                            git config user.name "Jenkins Bot"
                            git config user.email "jenkins@yourcompany.com"
                        '''

                        // Fetch all branches
                        sh 'git fetch origin'

                        // Switch to target branch
                        sh "git checkout ${env.TARGET_BRANCH}"
                        sh "git pull origin ${env.TARGET_BRANCH}"

                        // Switch to source branch to find commits
                        sh "git checkout ${env.SOURCE_BRANCH}"
                        sh "git pull origin ${env.SOURCE_BRANCH}"

                        // Find commits related to the PR
                        def prNumber = env.PR_NUMBER
                        def commitToSync = ""

                        if (prNumber != '0') {
                            // Try to find squash commit first
                            def squashCommit = sh(
                                script: "git log --oneline --grep='#${prNumber}' --format='%H' -n 1 || true",
                                returnStdout: true
                            ).trim()

                            if (squashCommit) {
                                commitToSync = squashCommit
                                echo "Found squash commit: ${commitToSync}"
                            } else {
                                // Try to find merge commit
                                def mergeCommit = sh(
                                    script: "git log --merges --grep='#${prNumber}' --format='%H' -n 1 || true",
                                    returnStdout: true
                                ).trim()

                                if (mergeCommit) {
                                    commitToSync = mergeCommit
                                    echo "Found merge commit: ${commitToSync}"
                                } else {
                                    error "Could not find commits for PR #${prNumber}"
                                }
                            }
                        } else {
                            // Fallback: use latest commit
                            commitToSync = sh(
                                script: 'git rev-parse HEAD',
                                returnStdout: true
                            ).trim()
                            echo "Using latest commit: ${commitToSync}"
                        }

                        // Switch to target branch and cherry-pick
                        sh "git checkout ${env.TARGET_BRANCH}"

                        // Check if it's a merge commit
                        def parentCount = sh(
                            script: "git rev-list --parents -n 1 ${commitToSync} | wc -w",
                            returnStdout: true
                        ).trim() as Integer
                        parentCount = parentCount - 1

                        if (parentCount > 1) {
                            echo "Merge commit detected, using -m 1"
                            sh "git cherry-pick -m 1 ${commitToSync}"
                        } else {
                            echo "Regular commit, cherry-picking normally"
                            sh "git cherry-pick ${commitToSync}"
                        }

                        // Push changes
                        sh "git push origin ${env.TARGET_BRANCH}"

                        env.SYNC_SUCCESS = 'true'
                        echo "✅ Successfully synced PR #${env.PR_NUMBER} to ${env.TARGET_BRANCH}"

                    } catch (Exception e) {
                        env.SYNC_SUCCESS = 'false'
                        env.SYNC_ERROR = e.getMessage()
                        echo "❌ Sync failed: ${e.getMessage()}"

                        // Abort any ongoing cherry-pick
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
                if (env.PR_NUMBER != '0') {
                    // Post success comment to GitHub PR
                    withCredentials([string(credentialsId: env.GITHUB_CREDENTIALS, variable: 'GITHUB_TOKEN')]) {
                        sh """
                            curl -X POST \\
                                -H "Authorization: token \$GITHUB_TOKEN" \\
                                -H "Content-Type: application/json" \\
                                -d '{
                                    "body": "✅ **Jenkins Auto-sync Successful!**\\n\\nChanges from PR #${env.PR_NUMBER} have been automatically synced to \`${env.TARGET_BRANCH}\` branch.\\n\\n🔗 [Jenkins Build](${env.BUILD_URL})"
                                }' \\
                                "https://api.github.com/repos/morphik/actions-test/issues/${env.PR_NUMBER}/comments"
                        """
                    }
                }
            }
        }

        failure {
            script {
                if (env.PR_NUMBER != '0') {
                    // Create GitHub issue for manual intervention
                    withCredentials([string(credentialsId: env.GITHUB_CREDENTIALS, variable: 'GITHUB_TOKEN')]) {
                        sh """
                            curl -X POST \\
                                -H "Authorization: token \$GITHUB_TOKEN" \\
                                -H "Content-Type: application/json" \\
                                -d '{
                                    "title": "🚨 Jenkins sync failed: PR #${env.PR_NUMBER} (${env.SOURCE_BRANCH} → ${env.TARGET_BRANCH})",
                                    "body": "**Automatic sync failed due to conflicts**\\n\\n**PR Details:**\\n- PR #${env.PR_NUMBER}: \\"${env.PR_TITLE}\\"\\n- Author: @${env.PR_AUTHOR}\\n- Source: \`${env.SOURCE_BRANCH}\`\\n- Target: \`${env.TARGET_BRANCH}\`\\n\\n**Error:** ${env.SYNC_ERROR ?: 'Unknown error'}\\n\\n**Jenkins Build:** ${env.BUILD_URL}\\n\\n**Manual sync required:**\\n\`\`\`bash\\ngit checkout ${env.TARGET_BRANCH}\\ngit pull origin ${env.TARGET_BRANCH}\\n\\n# Find and cherry-pick the commit\\ngit log --oneline ${env.SOURCE_BRANCH} --grep=\\"#${env.PR_NUMBER}\\"\\ngit cherry-pick <COMMIT_SHA>\\n\\n# Resolve conflicts, then:\\ngit add .\\ngit commit\\ngit push origin ${env.TARGET_BRANCH}\\n\`\`\`",
                                    "labels": ["jenkins-sync-failed", "needs-manual-intervention"]
                                }' \\
                                "https://api.github.com/repos/morphik/actions-test/issues"
                        """
                    }
                }
            }
        }

        always {
            // Clean up workspace
            cleanWs()
        }
    }
}