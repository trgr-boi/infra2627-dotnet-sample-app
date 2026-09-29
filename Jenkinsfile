pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        DB_CONTAINER = "todoapp-testdb-${env.BUILD_NUMBER}"
    }

    stages {

        stage('Preperation') {
            steps {
                sh '''
                    docker compose down --remove-orphans 2>/dev/null || true
                    docker rm -f todoapp todoappdb 2>/dev/null || true
                '''
            }
        }

        stage('Build') {
            steps {
                sh 'dotnet build -c Release dotnet-demo.slnx'
            }
        }

        stage('Test') {
            steps {
                sh '''
                    docker run -d --rm --name ${DB_CONTAINER} \
                        -e MARIADB_ROOT_PASSWORD=sekrit \
                        -e MARIADB_DATABASE=todo_test_db \
                        -e MARIADB_USER=todo_usr \
                        -e MARIADB_PASSWORD=letmeinplz \
                        -v "$PWD/TodoApp/schema.sql":/docker-entrypoint-initdb.d/schema.sql:ro \
                        -p 3306:3306 \
                        mariadb:11
                '''

                sh '''
                    for i in $(seq 1 30); do
                        if docker exec -e MARIADB_PWD=letmeinplz ${DB_CONTAINER} \
                                mariadb -h 127.0.0.1 -utodo_usr todo_test_db \
                                -e 'SELECT 1 FROM todos LIMIT 1;' >/dev/null 2>&1; then
                            exit 0
                        fi
                        sleep 2
                    done
                    echo "MariaDB test database was not ready in time" >&2
                    exit 1
                '''

                sh 'dotnet test -c Release --no-build --logger trx --results-directory TestResults'
            }
            post {
                always {
                    xunit(tools: [MSTest(deleteOutputFiles: false, pattern: '**/TestResults/*.trx')])
                    sh 'docker stop ${DB_CONTAINER} || true'
                }
            }
        }

        stage('Run') {
            steps {
                sh 'docker compose up -d --build'

                // test if app runs
                sh '''
                    curl -fsS --retry 15 --retry-delay 3 --retry-connrefused \
                        http://localhost:8080/ > /dev/null
                    echo "App is up: http://localhost:8080"
                '''
            }
        }
    }

    post {
        cleanup {
            cleanWs()
        }
    }
}
