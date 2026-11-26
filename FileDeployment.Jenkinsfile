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
                withBuildUser() {
                    script {
                        def buildUser = env.BUILD_USER ?: 'Unknown'
                        def branch = env.GIT_BRANCH ?: 'Unknown'

                        branch = branch.replaceFirst('^origin/', '')

                        currentBuild.displayName =
                            "#${BUILD_NUMBER} - ${buildUser} - ${branch}"

                        currentBuild.description =
                            "Triggered by: ${buildUser} | Git branch: ${branch}"

                        echo "Triggered by: ${buildUser}"
                        echo "Git branch: ${branch}"
                    }
                }
            }
        }

        stage('Validate Deployment Inputs') {
            steps {
                script {
                    if (!params.JIRA_ID?.trim()) {
                        error('JIRA ID is mandatory.')
                    }

                    if (params.GIT_REPOSITORY != 'coffee-shop') {
                        error('Unsupported Git repository selected.')
                    }

                    if (params.TARGET_SERVER != 'local-simulation') {
                        error('Unsupported deployment target selected.')
                    }

                    def requestedFiles = params.FILES_TO_DEPLOY
                        .readLines()
                        .collect { it.trim() }
                        .findAll { it }

                    if (requestedFiles.isEmpty()) {
                        error('Please provide at least one file to deploy.')
                    }

                    requestedFiles.each { filePath ->
                        if (filePath.startsWith('/') ||
                            filePath.tokenize('/').contains('..') ||
                            filePath.contains('\\')) {
                            error("Unsafe file path: ${filePath}")
                        }

                        if (!fileExists(filePath)) {
                            error("File not found in Git repository: ${filePath}")
                        }
                    }

                    writeFile(
                        file: 'deploy-files.txt',
                        text: requestedFiles.join('\n') + '\n'
                    )

                    sh '''
                        while IFS= read -r file; do
                            if [ ! -f "$file" ]; then
                                echo "ERROR: Not a regular file: $file"
                                exit 1
                            fi
                        done < deploy-files.txt
                    '''

                    echo "JIRA Ticket: ${params.JIRA_ID}"
                    echo "Repository: ${params.GIT_REPOSITORY}"
                    echo "Target: ${params.TARGET_SERVER}"
                    echo "Validated files:"
                    echo "${requestedFiles.join('\n')}"
                }
            }
        }
    }
}
