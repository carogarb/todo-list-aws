pipeline {
    agent any
    
    options {
        timestamps()
        skipDefaultCheckout()
        timeout(time: 5, unit: 'MINUTES')
    }

    environment {
        STAGE = "production"
        AWS_REGION = "us-east-1"  
        STACK_NAME = "todo-list-aws-${STAGE}"
        S3_BUCKET = "unirbucket1-cgb"
        GITHUB_TOKEN = credentials('github-token')
    }

    stages {
        stage('Get code') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    echo 'Getting the repo'
                    git branch: 'master', url: 'https://${GITHUB_TOKEN}@github.com/carogarb/todo-list-aws.git'
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Running sam build'
                sh 'sam build'

                echo 'Running sam deploy'
                sh '''
                    sam deploy \
                      --stack-name ${STACK_NAME} \
                      --s3-bucket ${S3_BUCKET} \
                      --region ${AWS_REGION} \
                      --capabilities CAPABILITY_IAM \
                      --no-confirm-changeset \
                      --no-fail-on-empty-changeset \
                      --parameter-overrides Stage=${STAGE} \
                      --on-failure DELETE
                '''
            }

        }

        stage('Rest Tests') {
            steps {
                script {
                    catchError(buildResult: 'ABORTED', stageResult: 'FAILURE') {
                        echo 'Export BASE_URL as a environment variable'
                        def BASE_URL = sh( script: "aws cloudformation describe-stacks --stack-name ${STACK_NAME} --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text",
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


