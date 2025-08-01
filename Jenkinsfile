pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'node24'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        BUILD_VERSION = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Git') {
            steps {
                git branch: 'main', url: 'https://github.com/mdjabiralhujaifa/note-todo-cicd-jabir.git'
            }
        }

        stage('SonarQube') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=Game \
                        -Dsonar.projectName=Game'''
                }
            }
        }

        stage('NPM Install') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP + Trivy FS') {
            steps {
                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'DP-Check'
                sh 'trivy fs . > trivyfs.txt'
                archiveArtifacts artifacts: 'trivyfs.txt', fingerprint: true
            }
        }

        stage('Docker Build + Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build -t jabiralhujaifa/todo:${BUILD_VERSION} ."
                        sh "docker push jabiralhujaifa/todo:${BUILD_VERSION}"
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image jabiralhujaifa/todo:${BUILD_VERSION} > trivy-image.txt"
                archiveArtifacts artifacts: 'trivy-image.txt', fingerprint: true
            }
        }

        stage('Kubernetes') {
            steps {
                script {
                    withKubeConfig(credentialsId: 'k8s') {
                        sh "kubectl set image deployment/todo-app todo=jabiralhujaifa/todo:${BUILD_VERSION}"
                    }
                }
            }
        }
    }
}
