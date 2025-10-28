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

@NonCPS
def hasCause(String name) {
  currentBuild.rawBuild.getCauses().any { c ->
    c.class.simpleName == name || c.class.name.endsWith("." + name)
  }
}
def isManualBuild()     { hasCause('UserIdCause') || hasCause('UserCause') }

pipeline {
    agent any

    environment {
        REPO_URL = 'https://github.com/morphik/actions-test.git'
        REPO = 'github.com/morphik/actions-test.git'
        ISSUES_URL = 'https://api.github.com/repos/morphik/actions-test/issues'
        GITHUB_CREDENTIALS = 'github-token'
        SKIP_SYNC = 'false'
        IS_PR_MERGE = 'false'
    }

    stages {

       stage('Detect PR merge (heuristic)') {
          steps {
            script {
              def parents = sh(script: "git cat-file -p HEAD | grep '^parent ' | wc -l", returnStdout: true).trim() as Integer
              def msg = sh(script: "git log -1 --pretty=%s", returnStdout: true).trim()

              // Extract PR number using shell (same method that works in your main pipeline)
              def prNumber = sh(
                script: """
                  echo '${msg}' | grep -oiE '#[0-9]+' | head -1 | sed 's/#//' || echo ''
                """,
                returnStdout: true
              ).trim()

              // PR merge is: merge commit (2+ parents) AND contains PR number
              env.IS_PR_MERGE = (parents >= 2 && prNumber != '') ? 'true' : 'false'
              env.PR_NUMBER = prNumber ?: '0'

              echo "Heuristic IS_PR_MERGE=${env.IS_PR_MERGE} (parents=${parents}, msg='${msg}')"
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


                echo "CHANGE_ID: #${env.CHANGE_ID}"
                echo "BRANCH_NAME ${env.BRANCH_NAME}"
                echo "TARGET ${env.CHANGE_TARGET}"
            }
        }

        stage('Manual stage') {
            when {
                expression { env.IS_PR_MERGE == 'false' && isManualBuild() }
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