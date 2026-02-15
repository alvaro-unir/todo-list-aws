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
                        git clone --branch develop https://$GH_USER:$GH_PAT@github.com/alvaro-unir/todo-list-aws.git .
                    '''
                }
                stash name: 'code', includes: '**'
            }
        }

        stage('Static Test') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    deleteDir()
                    unstash 'code'

                    sh '''
                        set +e

                        flake8 --exit-zero --format=pylint src > flake8.out
                        test -s flake8.out || echo "flake8: no issues found" > flake8.out

                        export PYTHONPATH=$WORKSPACE
                        bandit -r src -f custom --msg-template "{abspath}:{line}: [{test_id}] {msg}" > bandit.out
                        test -s bandit.out || echo "bandit: no issues found" > bandit.out

                        exit 0
                    '''

                    recordIssues tools: [
                        flake8(name: 'Flake8', pattern: 'flake8.out'),
                        pyLint(name: 'Bandit', pattern: 'bandit.out')
                    ]
                }
            }
        }

        stage('Deploy') {
            steps {
                deleteDir()
                unstash 'code'
                sh '''
                    set -e
                    sam build
                    sam deploy --config-env staging
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
                          --stack-name todo-list-aws-staging \
                          --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                          --output text)"

                        echo "BASE_URL=$BASE_URL"
                        pytest test/integration/todoApiTest.py
                    '''
                }
            }
        }

        stage('Promote') {
            steps {
                deleteDir()
                withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GH_USER', passwordVariable: 'GH_PAT')]) {
                    sh '''
                        set -e

                        git clone --branch master https://$GH_USER:$GH_PAT@github.com/alvaro-unir/todo-list-aws.git .
                        git config user.email "jenkins@local"
                        git config user.name  "jenkins"
                        git fetch origin develop
                        set +e
                        git merge --no-ff origin/develop -m "Release"
                        MERGE_RC=$?
                        set -e

                        if [ "$MERGE_RC" -ne 0 ]; then
                          echo "Merge conflict detected. Resolving in favor of origin/develop..."
                          git checkout --theirs -- .
                          git add -A
                          git diff --name-only --diff-filter=U | grep -q . && { echo "Unresolved conflicts remain"; exit 1; }
                          git commit -m "Release"
                        fi

                        git push origin master
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
