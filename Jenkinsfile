pipeline {
    agent any
    
    options {
        timestamps()
        skipDefaultCheckout()
        timeout(time: 5, unit: 'MINUTES')
    }

    environment {
        GITHUB_TOKEN = credentials('github-token')
    }

    stages {
        stage('Get code') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    echo 'Getting the repo'
                    git branch: 'master', url: 'https://${GITHUB_TOKEN}@github.com/carogarb/todo-list-aws.git'
                    sh '''
                        curl -o samconfig.toml https://raw.githubusercontent.com/carogarb/todo-list-aws-config/refs/heads/production/samconfig.toml
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Running sam build'
                sh 'sam build'

                echo 'Running sam deploy'
                sh '''
                    sam deploy --config-env production
                '''
            }

        }

        stage('Rest Tests') {
            steps {
                script {
                    catchError(buildResult: 'ABORTED', stageResult: 'FAILURE') {
                        echo 'Export BASE_URL as a environment variable'
                        def BASE_URL = sh( script: "aws cloudformation describe-stacks --stack-name todo-list-aws-production --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text",
                            returnStdout: true).trim()
                        echo "$BASE_URL"
                        
                        echo 'Running rest tests'
                        withEnv(["BASE_URL=${BASE_URL}"]) {
                            sh '''
                                export PYTHONPATH=.
                                export BASE_URL=$BASE_URL
                                pytest -m api_read --junitxml=result-rest.xml test//integration//todoApiTest.py || exit
                            '''
                        }
                        junit testResults: 'result-rest.xml', allowEmptyResults: false, skipPublishingChecks: true
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
    

}