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

                        mkdir -p .cd-config
                        git clone --depth 1 --branch production https://github.com/alvaro-unir/todo-list-aws-config.git .cd-config
                        cp .cd-config/samconfig.toml ./samconfig.toml
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
                    sam deploy --config-env default
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

                    set +e
                    pytest -q test/integration/todoApiTest.py -k "test_api_listtodos or test_api_gettodo" --junitxml=result-rest.xml
                    echo $? > pytest_rc
                    set -e
                '''

                junit testResults: 'result-rest.xml', allowEmptyResults: true

                sh '''
                    RC="$(cat pytest_rc)"
                    exit "$RC"
                '''
            }
        }
    }

    post {
        always { cleanWs() }
    }
}
