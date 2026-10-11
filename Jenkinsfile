// Full CI/CD pipeline for the Todo .Net app.
//
// This pipeline expects two env vars on the Jenkins controller, set in labenv/ansible/inventory/host_vars/infra010.yml:
//   TODO_APP_PORT     - host port the app container is published on
//   TODO_APP_HOST_IP  - IP of the Docker host, used by the acceptance test

pipeline {
    agent any

    stages {
        stage('Preparation') {
            steps {
                sh '''
                    set -eu

                    test -n "$TODO_APP_PORT" && test -n "$TODO_APP_HOST_IP" || {
                        echo 'FAIL: TODO_APP_PORT / TODO_APP_HOST_IP are not set on the Jenkins container (see host_vars/infra010.yml)'
                        exit 1
                    }

                    docker rm -f todoapp todoappdb 2> /dev/null || true
                '''
            }
        }
        stage('Unit tests') {
            steps {
                sh '''
                    set -eu

                    echo '=== Unit tests (dotnet test inside the SDK container) ==='
                    # These are integration tests. The fixture builds its config with
                    # AddJsonFile("appsettings.json"), a relative path resolved against the test
                    # host CWD (= the test output dir). Only TodoApp/appsettings.json lands there
                    # (transitive copy via the project reference; the Tests project has no Content
                    # item for its own file), so the effective connection string is
                    # Server=localhost:3306/todo_db. The test container therefore shares the
                    # network namespace of a temporary MariaDB container: localhost:3306 IS the
                    # database (and no env var override is possible: the config builder adds no
                    # environment variables). The workspace is not a path on the Docker host
                    # either, hence the image built from the workspace as build context.
                    printf '%s\n' \
                      'FROM mcr.microsoft.com/dotnet/sdk:10.0' \
                      'WORKDIR /src' \
                      'COPY . .' \
                      'ENTRYPOINT ["dotnet", "test"]' \
                      > Dockerfile.test
                    docker build -t dotnet-demo-test -f Dockerfile.test .

                    docker rm -f todoappdb-test 2> /dev/null || true
                    docker network create todo-net 2> /dev/null || true
                    trap 'docker rm -f todoappdb-test > /dev/null 2>&1 || true' EXIT
                    docker run -d --name todoappdb-test --network todo-net \
                      -e MARIADB_ROOT_PASSWORD=sekrit \
                      -e MARIADB_DATABASE=todo_db \
                      -e MARIADB_USER=todo_usr \
                      -e MARIADB_PASSWORD=letmeinplz \
                      mariadb:11

                    echo 'Waiting for the test database to become healthy...'
                    i=0
                    until docker exec todoappdb-test healthcheck.sh --connect --innodb_initialized; do
                      i=$((i + 1))
                      if [ "$i" -ge 30 ]; then
                        echo 'FAIL: test database not healthy after 60s'
                        exit 1
                      fi
                      sleep 2
                    done

                    echo '=== Initialize the test database schema ==='
                    docker exec -i todoappdb-test mariadb -utodo_usr -pletmeinplz todo_db < TodoApp/schema.sql

                    docker run --rm --network container:todoappdb-test dotnet-demo-test
                '''
            }
        }
        stage('Build & Deploy') {
            steps {
                sh '''
                    set -eu

                    echo '=== Build the app image (multi-stage Dockerfile from the app repo) ==='
                    docker build -t todoapp:latest TodoApp/

                    echo '=== Deploy MariaDB (data in the todoapp-db-data volume) ==='
                    docker rm -f todoapp todoappdb 2> /dev/null || true
                    docker volume create todoapp-db-data
                    # A user-defined network is needed so the app can resolve the database by container name (the default bridge has no DNS).
                    docker network create todo-net 2> /dev/null || true
                    docker run -d --name todoappdb --network todo-net \
                      -e MARIADB_ROOT_PASSWORD=sekrit \
                      -e MARIADB_DATABASE=todo_db \
                      -e MARIADB_USER=todo_usr \
                      -e MARIADB_PASSWORD=letmeinplz \
                      -v todoapp-db-data:/var/lib/mysql \
                      mariadb:11

                    echo 'Waiting for MariaDB to become healthy...'
                    i=0
                    until docker exec todoappdb healthcheck.sh --connect --innodb_initialized; do
                      i=$((i + 1))
                      if [ "$i" -ge 30 ]; then
                        echo 'FAIL: MariaDB not healthy after 60s'
                        exit 1
                      fi
                      sleep 2
                    done

                    echo '=== Initialize the database schema ==='
                    # No bind mount possible (workspace is inside the jenkins-data volume), so schema.sql is piped through docker exec stdin.
                    docker exec -i todoappdb mariadb -utodo_usr -pletmeinplz todo_db < TodoApp/schema.sql

                    echo "=== Deploy the app on port $TODO_APP_PORT ==="
                    docker run -d --name todoapp --network todo-net -p "$TODO_APP_PORT":8080 \
                      -e "ConnectionStrings__TodoDb=Server=todoappdb;Port=3306;Database=todo_db;User=todo_usr;Password=letmeinplz;" \
                      todoapp:latest

                    echo "PASS: todoapp deployed, published on port $TODO_APP_PORT"
                '''
            }
        }
        stage('Acceptance test') {
            steps {
                sh '''
                    set -eu

                    APP_URL="http://$TODO_APP_HOST_IP:$TODO_APP_PORT/"
                    RESPONSE=''

                    echo "Waiting for the app on $APP_URL ..."
                    for i in 1 2 3 4 5 6 7 8 9 10; do
                      if RESPONSE=$(curl -fs "$APP_URL" || true); then
                        break
                      fi
                      sleep 2
                    done

                    if [ -z "$RESPONSE" ]; then
                      echo "FAIL: could not reach $APP_URL"
                      exit 1
                    fi

                    if ! echo "$RESPONSE" | grep -qi todo; then
                      echo 'FAIL: unexpected response:'
                      echo "$RESPONSE"
                      exit 1
                    fi

                    echo "PASS: todo app is serving on $APP_URL"
                '''
            }
        }
    }
}
