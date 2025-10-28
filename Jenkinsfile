properties([
  pipelineTriggers([
    [$class: 'GenericTrigger',
      genericVariables: [
        [key: 'PR_ACTION',    value: '$.action'],
        [key: 'PR_MERGED',    value: '$.pull_request.merged'],
        [key: 'PR_TARGET',    value: '$.pull_request.base.ref'],
        [key: 'PR_NUMBER',    value: '$.pull_request.number'],
        [key: 'PR_MERGE_SHA', value: '$.pull_request.merge_commit_sha'],
      ],
      // token musi zgadzać się z URLem webhooka
      token: '447e3c9d-b10b-4d56-a977-7b2a3c54975d',
      // Tylko zamknięty PR z merged=true
      regexpFilterText: '$PR_ACTION $PR_MERGED',
      regexpFilterExpression: 'closed true',
      printContributedVariables: true,
      printPostContent: false
    ]
  ])
])


pipeline {
    agent any

    environment {
        REPO_URL = 'https://github.com/morphik/actions-test.git'
        REPO = 'github.com/morphik/actions-test.git'
        ISSUES_URL = 'https://api.github.com/repos/morphik/actions-test/issues'
        GITHUB_CREDENTIALS = 'github-token'
        SKIP_SYNC = 'false'
    }

    stages {

       stage('Detect PR merge (heuristic)') {
          steps {
            script {
              def parents = sh(script: "git cat-file -p HEAD | grep '^parent ' | wc -l", returnStdout: true).trim() as Integer
              def msg = sh(script: "git log -1 --pretty=%s", returnStdout: true).trim()

              echo "=== DEBUG ==="

              def prNumber = sh(
                script: "echo '${msg}' | grep -oiE '#[0-9]+' | head -1 | sed 's/#//' || echo ''",
                returnStdout: true
              ).trim()

              echo "parents = '${parents}' (class: ${parents.class.name})"
              echo "prNumber = '${prNumber}' (class: ${prNumber.class.name})"
              echo "parents >= 2 = ${parents >= 2}"
              echo "prNumber != '' = ${prNumber != ''}"
              echo "prNumber.isEmpty() = ${prNumber.isEmpty()}"

              // Use explicit if-else
              env.PR_NUMBER = prNumber
              if (parents >= 2 && !prNumber.isEmpty()) {
                env.IS_PR_MERGE = 'true'
              } else {
                env.IS_PR_MERGE = 'false'
              }

              echo "IS_PR_MERGE=${env.IS_PR_MERGE}, PR_NUMBER=${env.PR_NUMBER}"
            }
          }
        }

        stage('Debug Stage') {
            steps {

                echo "PR_ACTION: ${env.PR_ACTION}"
                echo "PR_MERGED: ${env.PR_MERGED}"
                echo "PR_TARGET: ${env.PR_TARGET}"
                echo "PR_NUMBER: ${env.PR_NUMBER}"
                echo "PR_MERGE_SHA: ${env.PR_MERGE_SHA}"

                echo "IS_PR_MERGE: ${env.IS_PR_MERGE }"

                echo "CHANGE_ID: #${env.CHANGE_ID}"
                echo "BRANCH_NAME ${env.BRANCH_NAME}"
                echo "TARGET ${env.CHANGE_TARGET}"
            }
        }

        stage('Manual stage') {
            when {
                allOf {
                    expression { env.IS_PR_MERGE == 'false' }
                    triggeredBy 'UserIdCause'
                }

                beforeAgent true
            }
            steps { echo 'Start ręczny → uruchamiam ten stage' }
        }

        stage('Tylko PR') {
            when {
                expression { env.IS_PR_MERGE == 'true' }
                beforeAgent true
            }
            steps { echo 'Change Request → uruchamiam PR' }
        }
    }
}