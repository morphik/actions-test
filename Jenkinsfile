pipeline {
    agent any

    // No triggers block - will be triggered by webhook only
    // Configure in Jenkins job UI: Build Triggers -> "GitHub hook trigger for GITScm polling"

    environment {
        REPO_URL = 'https://github.com/morphik/actions-test.git'
        GITHUB_CREDENTIALS = 'github-token'
        SOURCE_BRANCH = 'release1.3'
        TARGET_BRANCH = 'release1.4'
    }

    stages {
        stage('Verify PR Merged') {
            steps {
                script {
                    // Check if this was triggered by a PR merge
                    def prMerged = env.ghprbPullId && env.ghprbActualCommit

                    if (!prMerged) {
                        echo "⚠️ This pipeline should only run on PR merge events"
                        echo "Skipping execution..."
                        currentBuild.result = 'NOT_BUILT'
                        error("Pipeline not triggered by PR merge - skipping")
                    }

                    echo "✅ Verified: Triggered by PR merge"
                }
            }
        }

        stage('Checkout Repository') {
            steps {
                script {
                    deleteDir()

                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${env.SOURCE_BRANCH}"]],
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [
                            [$class: 'CloneOption', depth: 0, noTags: false, reference: '', shallow: false]
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
                    env.PR_NUMBER = env.ghprbPullId ?: env.GHPRB_PULL_ID ?: '0'
                    env.PR_TITLE = env.ghprbPullTitle ?: env.GHPRB_PULL_TITLE ?: 'Unknown'
                    env.PR_AUTHOR = env.ghprbPullAuthorLogin ?: env.GHPRB_PULL_AUTHOR_LOGIN ?: 'unknown'
                    env.PR_TARGET_BRANCH = env.ghprbTargetBranch ?: env.GHPRB_TARGET_BRANCH ?: env.SOURCE_BRANCH

                    echo "=== PR Merge Information ==="
                    echo "PR Number: #${env.PR_NUMBER}"
                    echo "PR Title: ${env.PR_TITLE}"
                    echo "Author: ${env.PR_AUTHOR}"
                    echo "Target Branch: ${env.PR_TARGET_BRANCH}"
                    echo "Sync: ${env.SOURCE_BRANCH} → ${env.TARGET_BRANCH}"
                    echo "============================"
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

                        sh "git checkout ${env.TARGET_BRANCH}"
                        sh "git pull origin ${env.TARGET_BRANCH}"

                        sh "git checkout ${env.SOURCE_BRANCH}"
                        sh "git pull origin ${env.SOURCE_BRANCH}"

                        def prNumber = env.PR_NUMBER
                        def commitToSync = ""

                        if (prNumber != '0') {
                            echo "🔍 Looking for PR #${prNumber} commits..."

                            def squashCommit = sh(
                                script: "git log --oneline --grep='#${prNumber}' --format='%H' -n 1 || true",
                                returnStdout: true
                            ).trim()

                            if (squashCommit) {
                                commitToSync = squashCommit
                                echo "✅ Found squash commit: ${commitToSync}"
                            } else {
                                def mergeCommit = sh(
                                    script: "git log --merges --grep='#${prNumber}' --format='%H' -n 1 || true",
                                    returnStdout: true
                                ).trim()

                                if (mergeCommit) {
                                    commitToSync = mergeCommit
                                    echo "✅ Found merge commit: ${commitToSync}"
                                } else {
                                    echo "⚠️ Could not find PR #${prNumber} commits, using HEAD"
                                    commitToSync = sh(
                                        script: 'git rev-parse HEAD',
                                        returnStdout: true
                                    ).trim()
                                }
                            }
                        } else {
                            commitToSync = sh(
                                script: 'git rev-parse HEAD',
                                returnStdout: true
                            ).trim()
                            echo "📝 Using latest commit: ${commitToSync}"
                        }

                        def commitMsg = sh(
                            script: "git log --oneline -n 1 ${commitToSync}",
                            returnStdout: true
                        ).trim()
                        echo "📋 Commit to sync: ${commitMsg}"

                        sh "git checkout ${env.TARGET_BRANCH}"

                        def parentCount = sh(
                            script: "git rev-list --parents -n 1 ${commitToSync} | wc -w",
                            returnStdout: true
                        ).trim() as Integer
                        parentCount = parentCount - 1

                        if (parentCount > 1) {
                            echo "🔀 Merge commit detected (${parentCount} parents), using -m 1"
                            sh "git cherry-pick -m 1 ${commitToSync}"
                        } else {
                            echo "📝 Regular commit, cherry-picking"
                            sh "git cherry-pick ${commitToSync}"
                        }

                        echo "📤 Pushing to ${env.TARGET_BRANCH}..."
                        sh "git push origin ${env.TARGET_BRANCH}"

                        env.SYNC_SUCCESS = 'true'
                        env.SYNCED_COMMIT = commitToSync
                        echo "✅ Successfully synced PR #${env.PR_NUMBER} to ${env.TARGET_BRANCH}"

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
                echo "✅ Branch sync completed successfully"

                if (env.PR_NUMBER != '0') {
                    try {
                        withCredentials([string(credentialsId: env.GITHUB_CREDENTIALS, variable: 'GITHUB_TOKEN')]) {
                            sh """
                                curl -X POST \\
                                    -H "Authorization: token \$GITHUB_TOKEN" \\
                                    -H "Content-Type: application/json" \\
                                    -d '{"body": "✅ **Jenkins Auto-sync Successful!**\\n\\nChanges from PR #${env.PR_NUMBER} have been automatically synced to ${env.TARGET_BRANCH} branch.\\n\\n🔗 [Jenkins Build](${env.BUILD_URL})\\n📝 Commit: ${env.SYNCED_COMMIT}"}' \\
                                    "https://api.github.com/repos/morphik/actions-test/issues/${env.PR_NUMBER}/comments"
                            """
                        }
                        echo "✅ Posted success comment to PR #${env.PR_NUMBER}"
                    } catch (Exception e) {
                        echo "⚠️ Could not post comment: ${e.getMessage()}"
                    }
                }
            }
        }

        failure {
            script {
                echo "❌ Branch sync failed"

                if (env.PR_NUMBER != '0') {
                    try {
                        withCredentials([string(credentialsId: env.GITHUB_CREDENTIALS, variable: 'GITHUB_TOKEN')]) {
                            sh """
                                curl -X POST \\
                                    -H "Authorization: token \$GITHUB_TOKEN" \\
                                    -H "Content-Type: application/json" \\
                                    -d '{"title": "🚨 Jenkins sync failed: PR #${env.PR_NUMBER} (${env.SOURCE_BRANCH} → ${env.TARGET_BRANCH})", "body": "**Automatic sync failed**\\n\\n**PR Details:**\\n- PR #${env.PR_NUMBER}: ${env.PR_TITLE}\\n- Author: @${env.PR_AUTHOR}\\n- Source: ${env.SOURCE_BRANCH}\\n- Target: ${env.TARGET_BRANCH}\\n\\n**Error:** ${env.SYNC_ERROR ?: 'Unknown error'}\\n\\n**Jenkins Build:** ${env.BUILD_URL}\\n\\n**Manual sync:**\\n\\ngit checkout ${env.TARGET_BRANCH}\\ngit pull\\ngit log --grep=#${env.PR_NUMBER} ${env.SOURCE_BRANCH}\\ngit cherry-pick COMMIT_SHA\\ngit push origin ${env.TARGET_BRANCH}", "labels": ["jenkins-sync-failed", "manual-intervention"]}' \\
                                    "https://api.github.com/repos/morphik/actions-test/issues"
                            """
                        }
                        echo "✅ Created GitHub issue for sync failure"
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