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

        stage('Debug Stage') {
            steps {
                echo "CHANGE_ID: #${env.CHANGE_ID}"
                echo "BRANCH_NAME ${env.BRANCH_NAME}"
                echo "TARGET ${env.CHANGE_TARGET}"
            }
        }

        stage('Manual stage') {
            when {
                not { changeRequest() }
                beforeAgent true
            }
            steps { echo 'Start ręczny → uruchamiam ten stage' }
        }

        stage('Tylko PR') {
            when {
                changeRequest()
                beforeAgent true
            }
            steps { echo 'Change Request → uruchamiam PR' }
        }
    }
}