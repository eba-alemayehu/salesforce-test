pipeline {
    agent any

    parameters {
        string(name: 'GIT_COMMIT_ID', defaultValue: '', description: 'Enter the starting Git commit ID')
    }

    triggers {
        pullSCM "* * * * *"
    }

    environment {
        SF_AUTH_URL = credentials('SALESFORCE_AUTH_URL') 
        SF_USERNAME = 'sf-username'
    }

    stages {
        stage('Checkout Code') {
            steps {
                script {
                    sh 'git clone https://github.com/eba-alemayehu/salesforce-test.git salesforce-metadata'
                    dir('salesforce-metadata') {
                        sh 'git checkout main'
                        sh 'git pull origin main'
                    }
                }
            }
        }

        
        stage('Find Commits to Deploy') {
            steps {
                script {
                    dir('salesforce-metadata') {
                        def commitList = sh(script: "git log --pretty=format:'%H' ${params.START_COMMIT}..HEAD", returnStdout: true).trim().split("\n").reverse()

                        if (commitList.size() == 0) {
                            error "No new commits to deploy from ${params.START_COMMIT}."
                        }

                        echo "Commits to be deployed:\n${commitList.join('\n')}"
                        writeFile(file: 'commitList.txt', text: commitList.join('\n'))
                    }
                }
            }
        }

        stage('Deploy Commits One by One') {
            steps {
                script {
                    dir('salesforce-metadata') {
                        def commits = readFile('commitList.txt').trim().split("\n")

                        for (commit in commits) {
                            echo "Processing commit: ${commit}"

                            def changedFiles = sh(script: "git diff-tree --no-commit-id --name-only -r ${commit}", returnStdout: true).trim()

                            if (!changedFiles) {
                                echo "No deployable changes in ${commit}. Skipping..."
                                continue
                            }

                            echo "Changed files in commit ${commit}:\n${changedFiles}"

                            sh 'mkdir -p deploy'
                            sh "cp --parents ${changedFiles.replaceAll('\n', ' ')} deploy/ || true"

                            sh "sf org login sfdx-url --sfdx-url-file=${SF_AUTH_URL} --set-default"

                        
                            def testResult = sh(script: "sf apex run test --result-format junit --target-org ${SF_USERNAME}", returnStatus: true)

                            if (testResult != 0) {
                                error "Apex tests failed for commit ${commit}. Stopping deployment!"
                            }

                            sh "sf project deploy start --source-dir deploy --target-org ${SF_USERNAME} --wait 10"

                        
                            sh 'rm -rf deploy'
                        }
                    }
                }
            }
        }

    
         stage('Deploy Commits One by One with Tests') {
            steps {
                script {
                    dir('salesforce-metadata') {
                        def commits = readFile('commitList.txt').trim().split("\n")

                        for (commit in commits) {
                            echo "Processing commit: ${commit}"

                            def changedFiles = sh(script: "git diff-tree --no-commit-id --name-only -r ${commit}", returnStdout: true).trim()

                            if (!changedFiles) {
                                echo "No deployable changes in ${commit}. Skipping..."
                                continue
                            }

                            echo "Changed files in commit ${commit}:\n${changedFiles}"

                            sh 'mkdir -p deploy'
                            sh "cp --parents ${changedFiles.replaceAll('\n', ' ')} deploy/ || true"

                            sh "sf org login sfdx-url --sfdx-url-file=${SF_AUTH_URL} --set-default"

                            echo "Running Apex tests before deploying commit ${commit}..."
                            def testResult = sh(script: "sf apex run test --result-format junit --target-org ${SF_USERNAME} --wait 10", returnStatus: true)

                            if (testResult != 0) {
                                error "Apex tests failed for commit ${commit}. Stopping deployment!"
                            }

                            echo "Deploying commit ${commit}..."
                            sh "sf project deploy start --source-dir deploy --target-org ${SF_USERNAME} --wait 10"

                            def deployStatus = sh(script: "sf project deploy report --wait 10 --target-org ${SF_USERNAME}", returnStatus: true)

                            if (deployStatus != 0) {
                                error "Deployment failed for commit ${commit}. Stopping!"
                            }

                            sh 'rm -rf deploy'
                        }
                    }
                }
            }
        }

        stage('Post Deployment Cleanup') {
            steps {
                script {
                    sh "rm -rf salesforce-metadata"
                }
            }
        }
    }
}