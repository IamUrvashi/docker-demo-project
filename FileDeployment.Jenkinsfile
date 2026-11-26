pipeline {
    agent any

    parameters {
        string(
            name: 'JIRA_ID',
            defaultValue: '',
            description: 'Enter the approved JIRA ticket ID'
        )

        choice(
            name: 'GIT_REPOSITORY',
            choices: ['coffee-shop'],
            description: 'Select the Git repository'
        )

        choice(
            name: 'TARGET_SERVER',
            choices: ['local-simulation'],
            description: 'Select the deployment target'
        )

        text(
            name: 'FILES_TO_DEPLOY',
            defaultValue: 'inventory_service/service.py',
            description: 'Enter repository-relative file paths, one per line'
        )
    }

    stages {
        stage('Build Information') {
            steps {
                wrap([$class: 'BuildUser']) {
                    script {
                        def buildUser = env.BUILD_USER ?: 'Unknown'
                        def branch = env.GIT_BRANCH ?: 'Unknown'

                        branch = branch.replaceFirst('^origin/', '')

                        currentBuild.displayName =
                            "#${env.BUILD_NUMBER} - ${buildUser} - ${branch}"

                        currentBuild.description =
                            "Triggered by ${buildUser} | Branch: ${branch}"

                        echo "Build Number: ${env.BUILD_NUMBER}"
                        echo "Build User: ${buildUser}"
                        echo "Git Branch: ${branch}"
                    }
                }
            }
        }

        stage('Validate Deployment Inputs') {
            steps {
                echo "JIRA Ticket: ${params.JIRA_ID}"
                echo "Repository: ${params.GIT_REPOSITORY}"
                echo "Target: ${params.TARGET_SERVER}"
                echo 'Files requested:'
                echo "${params.FILES_TO_DEPLOY}"
            }
        }
    }
}
