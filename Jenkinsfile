node {
    stage('Preparation') {
        catchError(buildResult: 'SUCCESS') {
            sh 'docker rm -f todoapp todoappdb'
        }
    }
    stage('Build') {
        build 'BuildDotnetApp'
    }
    stage('Results') {
        build 'TestDotnetApp'
    }
}
