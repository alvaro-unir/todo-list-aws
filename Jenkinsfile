pipeline {
    agent any

    options { skipDefaultCheckout() }

    stages {

        stage('Get Code') {
            steps {
                deleteDir()
                checkout scm
                stash name: 'code', includes: '**'
            }
        }

        stage('Deploy') {
            steps {
                deleteDir()
                unstash 'code'
                sh '''
                    set -e
                    sam build
                    sam deploy --config-env production
                '''
            }
        }

        stage('Rest Test') {
            steps {
                deleteDir()
                unstash 'code'
                sh '''
                    set -e

                    export BASE_URL="$(aws cloudformation describe-stacks \
                      --stack-name todo-list-aws-production \
                      --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                      --output text)"

                    echo "BASE_URL=$BASE_URL"

                    pytest test/integration/todoApiTest.py -k "test_api_listtodos or test_api_gettodo"
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