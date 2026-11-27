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

        stage('Deploy Files') {
            steps {
                script {
                    writeFile(
                        file: 'deploy.sh',
                        text: '''#!/bin/sh
set -eu

SOURCE_ROOT="$WORKSPACE"
DEPLOY_ROOT="/deployment/deployment-target"
BACKUP_ROOT="/deployment/backups"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)

while IFS= read -r FILE_PATH || [ -n "$FILE_PATH" ]; do
    [ -z "$FILE_PATH" ] && continue

    SOURCE="$SOURCE_ROOT/$FILE_PATH"
    TARGET="$DEPLOY_ROOT/$FILE_PATH"
    BACKUP_DIR="$BACKUP_ROOT/$(dirname "$FILE_PATH")"

    echo "===================================="
    echo "Deploying: $FILE_PATH"

    if [ ! -f "$SOURCE" ]; then
        echo "ERROR: Source file does not exist: $SOURCE"
        exit 1
    fi

    mkdir -p "$(dirname "$TARGET")"

    if [ -L "$TARGET" ]; then
        echo "ERROR: Target is a symbolic link: $TARGET"
        exit 1
    fi

    if [ ! -f "$TARGET" ]; then
        echo "ERROR: Target file does not exist: $TARGET"
        exit 1
    fi

    mkdir -p "$BACKUP_DIR"

    ORIGINAL_MODE=$(stat -c '%a' "$TARGET")
    BACKUP="$BACKUP_DIR/$(basename "$FILE_PATH").$TIMESTAMP.bak"

    if [ -e "$BACKUP" ]; then
        echo "ERROR: Backup already exists: $BACKUP"
        exit 1
    fi

    cp -p "$TARGET" "$BACKUP"

    echo "Backup created: $BACKUP"
    echo "Original target permissions: $ORIGINAL_MODE"

    cp "$SOURCE" "$TARGET"
    chmod "$ORIGINAL_MODE" "$TARGET"

    SOURCE_HASH=$(sha256sum "$SOURCE" | awk '{print $1}')
    TARGET_HASH=$(sha256sum "$TARGET" | awk '{print $1}')
    TARGET_MODE=$(stat -c '%a' "$TARGET")

    echo "Source SHA256: $SOURCE_HASH"
    echo "Target SHA256: $TARGET_HASH"
    echo "Target permissions: $TARGET_MODE"

    if [ "$SOURCE_HASH" != "$TARGET_HASH" ]; then
        echo "ERROR: Checksum verification failed."
        exit 1
    fi

    if [ "$TARGET_MODE" != "$ORIGINAL_MODE" ]; then
        echo "ERROR: Permission verification failed."
        exit 1
    fi

    echo "SUCCESS: Deployment and verification completed."

done < "$SOURCE_ROOT/deploy-files.txt"
'''
                    )

                    sh '''
                        docker run --rm \
                            --volumes-from jenkins \
                            -v /Users/ughangare/File-Deployment-Project:/deployment \
                            -e WORKSPACE="$WORKSPACE" \
                            alpine:3.20 \
                            sh "$WORKSPACE/deploy.sh"
                    '''
                }
            }
        }
    }
}
