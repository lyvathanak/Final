pipeline {
    // Agent 'none' means the main pipeline doesn't use a specific environment,
    // but each stage will define its own.
    agent none

    // Tools section specifies tools to be available in the PATH.
    // Requires Ansible plugin to be configured in Manage Jenkins -> Tools.
    tools {
        ansible 'ansible' // 'ansible' must match the name you give it in Jenkins config
    }

    // Triggers define how the pipeline is started automatically.
    triggers {
        // Poll SCM (Source Code Management) every 5 minutes for changes.
        pollSCM('H/5 * * * *')
    }

    // Parameters allow for manual builds with options, not used here but good practice.
    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to build')
    }

    stages {
        // STAGE 1: Prepare the environment by cleaning the workspace
        stage('Prepare Workspace') {
            agent any
            steps {
                script {
                    echo "Cleaning up workspace before build..."
                    // The cleanWs() step deletes all files from the previous build.
                    cleanWs()
                }
            }
        }

        // STAGE 2: Checkout the code from GitHub
        stage('Checkout Code') {
            agent any
            steps {
                script {
                    echo "Checking out code from branch: ${params.BRANCH}"
                    // The git step checks out the code from your repository.
                    git branch: params.BRANCH, url: 'https://github.com/lyvathanak/Final.git'
                }
            }
        }

        // STAGE 3: Build & Test the application inside a Docker container
        stage('Build & Test') {
            // This stage runs inside a Docker container based on the specified image.
            agent {
                docker {
                    image 'webdevops/php-nginx:8.2-alpine'
                    // The user is root inside this container, which has permission for all commands.
                    args '-u root'
                }
            }
            steps {
                script {
                    echo "Starting build and test process..."
                    // Sequence of commands to build and test the application
                    sh '''
                        # Install git to add safe directory, and nodejs/npm
                        apk add --no-cache git nodejs npm

                        # Add git safe directory to avoid ownership errors
                        git config --global --add safe.directory /var/jenkins_home/workspace/Final_main

                        # Install composer dependencies, including dev for testing
                        composer install --no-interaction --optimize-autoloader

                        # Create environment file and generate app key
                        cp .env.example .env
                        php artisan key:generate

                        # Install NPM dependencies and build assets
                        npm install
                        npm run build

                        # Set up the test database and run tests
                        touch database/database.sqlite
                        php artisan migrate --force
                        php artisan test > test-output.txt
                    '''
                    // Stash the test output file to use in the post-build actions
                    stash name: 'test-output', includes: 'test-output.txt'
                }
            }
        }

        // STAGE 4: Deploy the application using Ansible
        stage('Deploy with Ansible') {
            agent any
            steps {
                script {
                    echo "Running Ansible playbook to deploy..."
                    // Execute the ansible-playbook command.
                    // Jenkins needs to be able to find the kubectl command.
                    sh 'ansible-playbook deploy-playbook.yaml'
                }
            }
        }
    }

    // POST-BUILD ACTIONS: These run after all stages are complete.
    post {
        // This 'always' block runs regardless of build success or failure.
        always {
            script {
                echo "Build finished. Archiving artifacts..."
                // Archive the database backup and the built output files for download.
                archiveArtifacts artifacts: 'backup.sql, test-output.txt', followSymlinks: false
            }
        }
        // This 'failure' block only runs if the build fails at any stage.
        failure {
            script {
                echo "Build FAILED. Sending notification email..."
                // Get the email of the person who made the last commit
                def commitAuthorEmail = sh(returnStdout: true, script: 'git log -1 --pretty=format:%ae').trim()
                // Send an email notification
                emailext (
                    subject: "BUILD FAILED: Job '${env.JOB_NAME}' - Build #${env.BUILD_NUMBER}",
                    body: """<p>A build has failed for job: <b>${env.JOB_NAME}</b></p>
                           <p>Build Number: ${env.BUILD_NUMBER}</p>
                           <p>Committer: ${commitAuthorEmail}</p>
                           <p>Check console output at: <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>""",
                    to: "srengty@gmail.com, ${commitAuthorEmail}"
                )
            }
        }
    }
}

