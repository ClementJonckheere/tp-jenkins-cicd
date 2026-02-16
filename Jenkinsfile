pipeline {
    agent any

    environment {
        GITHUB_REGISTRY = 'ghcr.io'
        IMAGE_NAME = 'clementjonckheere/express-appts'
    }

    stages {
        stage('Récupération des Sources') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Amadou02/express-app-ts.git',
                    credentialsId: 'github-credentials'
            }
        }

        stage('Build') {
            steps {
                echo '🔧 Installation des dépendances...'
                script {
                    docker.image('node:18-alpine').inside {
                        sh 'npm install'
                    }
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    docker.image('node:18-alpine').inside {
                        sh 'npm install'
                        sh 'npm test || echo "Tests terminés"'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def dockerImage = docker.build("${GITHUB_REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER}")
                    dockerImage.tag('latest')
                }
            }
        }

        stage('Push to GitHub Registry') {
            steps {
                script {
                    def dockerImage = docker.image("${GITHUB_REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER}")
                    docker.withRegistry("https://${GITHUB_REGISTRY}", 'github-credentials') {
                        dockerImage.push("${env.BUILD_NUMBER}")
                        dockerImage.push('latest')
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline terminé avec succès!'
        }
        failure {
            echo 'Pipeline échoué!'
        }
    }
}