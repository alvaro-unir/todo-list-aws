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

                        # Repo app (develop)
                        git clone --branch develop https://$GH_USER:$GH_PAT@github.com/alvaro-unir/todo-list-aws.git .

                        # Repo config (staging) -> samconfig.toml
                        mkdir -p .ci-config
                        git clone --depth 1 --branch staging https://github.com/alvaro-unir/todo-list-aws-config.git .ci-config
                        cp .ci-config/samconfig.toml ./samconfig.toml
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

                        flake8 --exit-zero --format=pylint src > flake8.out 2>&1
                        test -s flake8.out || echo "flake8: no issues found" > flake8.out

                        export PYTHONPATH="$WORKSPACE"
                        bandit -r src -f custom --msg-template "{abspath}:{line}: [{test_id}] {msg}" > bandit.out 2>&1
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
                    # En el repo config (rama staging), el entorno "default" debe apuntar a staging
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
                      --stack-name todo-list-aws-staging \
                      --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                      --output text)"

                    echo "BASE_URL=$BASE_URL"

                    set +e
                    pytest test/integration/todoApiTest.py --junitxml=result-rest.xml
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
                          git commit -m "Release (auto-resolve)"
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
