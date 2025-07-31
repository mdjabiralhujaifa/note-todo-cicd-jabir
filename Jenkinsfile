pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'node24'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('pull from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/mdjabiralhujaifa/note-todo-cicd-jabir.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=Game \
                        -Dsonar.projectName=Game'''
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
                archiveArtifacts artifacts: 'trivyfs.txt', fingerprint: true
            }
        }

        stage("Docker Build & Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build -t todo ."
                        sh "docker tag todo jabiralhujaifa/todo:latest"
                        sh "docker push jabiralhujaifa/todo:latest"
                        
                    }
                }
            }
        }

        stage("TRIVY Image Scan") {
            steps {
                sh "trivy image jabiralhujaifa/todo:latest > trivy-image-scan.txt"
                archiveArtifacts artifacts: 'trivy-image-scan.txt', fingerprint: true
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker rm -f todo || true'
                sh 'docker run -d --name todo -p 3000:3000 jabiralhujaifa/todo:latest'
                
            }
        }
        stage('Deploy to kubernets') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                        sh 'kubectl apply -f deployment.yaml'
                    }
                }
            }
        }
    }
}
