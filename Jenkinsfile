pipeline {
    agent any

    options { skipDefaultCheckout() }

    stages {

        stage('Get Code') {
            steps {
                deleteDir()
                withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GH_USER', passwordVariable: 'GH_PAT')]) {
                    sh '''
                        set -e
                        git clone --branch master https://$GH_USER:$GH_PAT@github.com/alvaro-unir/todo-list-aws.git .
                    '''
                }
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
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    deleteDir()
                    unstash 'code'
                    sh '''
                        set -e

                        export BASE_URL="$(aws cloudformation describe-stacks \
                          --stack-name todo-list-aws-production \
                          --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                          --output text)"

                        echo "BASE_URL=$BASE_URL"

                        # Solo tests 1 y 3 (lectura): listtodos y gettodo
                        pytest -q test/integration/todoApiTest.py -k "test_api_listtodos or test_api_gettodo"
                    '''
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
