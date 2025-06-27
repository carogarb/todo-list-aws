pipeline {
    agent any
    
    options {
        timestamps()
        skipDefaultCheckout()
        timeout(time: 5, unit: 'MINUTES')
    }

    environment {
        STAGE = "staging"
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
                    git branch: 'develop', url: 'https://${GITHUB_TOKEN}@github.com/carogarb/todo-list-aws.git'
                }
            }
        }

        stage('Static code analysis') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    echo 'Running flake8'
                    sh '''
                        export PYTHONPATH=.
                        flake8 --format=pylint --exit-zero src>flake8.out
                    '''

                    recordIssues tools:[flake8(name:'Flake8', pattern:'flake8.out')]
                }    

                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    echo 'Running bandit'
                    sh '''
                        export PYTHONPATH=.
                        bandit --exit-zero -r ./src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                    '''

                    recordIssues tools:[pyLint(name:'Bandit', pattern:'bandit.out')]
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
                                pytest --junitxml=result-rest.xml test//integration//todoApiTest.py || exit
                            '''
                        }
                        junit testResults: 'result-rest.xml', allowEmptyResults: false, skipPublishingChecks: true
                    }
                }
            }
        }
        
        stage('Promote') {
            when {
                expression { currentBuild.resultIsBetterOrEqualTo('SUCCESS') }
            }
            steps {
                echo 'Promoting to PROD environment'
                sh '''
                        git checkout master
                        git merge -Xours develop
                        git push https://${GITHUB_TOKEN}@github.com/carogarb/todo-list-aws.git master
                '''  
            }
        }

    }
    
    post {
        always {
            cleanWs()
        }
    }
}


