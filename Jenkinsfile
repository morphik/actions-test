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