pipeline {
    agent any

    stages {
        stage('Build and Run') {
            steps {
                sh '''
                cp -r /node-todo-cicd /tmp/node-todo-copy
                cd /tmp/node-todo-copy

                docker build . -t node-todo-image
                docker stop node-todo-container || true
                docker rm node-todo-container || true
                docker run -d --name node-todo-container -p 8000:8000 node-todo-image
                '''
            }
        }
    }
}
