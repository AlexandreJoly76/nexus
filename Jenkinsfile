pipeline {
    agent any

    triggers {
        pollSCM('* * * * *')
    }

    tools {
        maven 'maven-3'
        jdk 'jdk-17'
    }

    environment {
        FINAL_EMAIL = "${env.DEVOPS_EMAIL ?: 'ton.email@gmail.com'}"
        NEXUS_CRED = "1"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test Backend') {
            steps {
                dir('microservices/user-service') {
                    sh 'mvn clean verify'
                }
            }
        }

        stage('Deploy to Nexus (Maven)') {
            steps {
                withCredentials([usernamePassword(credentialsId: "1",
                usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                dir('microservices/user-service') {
                    """
                    sh 'mvn deploy -s ../../settings.xml'
                    -Dnexus.user=$NEXUS_USER \
                    -Dnexus.pass=$NEXUS_PASS
                    """
                }
            }
            }
        }

        stage('Build Frontend') {
            steps {
                dir('frontend/buy01-web') {
                    sh 'npm install'
                    sh 'npm run build'
                }
            }
        }

        stage('Deploy (Docker)') {
            steps {
                dir('infrastructure') {
                    script {
                        sh 'docker compose down || true'
                        sh 'docker compose up -d --build'
                    }
                }
            }
        }
    }

    post {
        success {
            mail to: "${env.FINAL_EMAIL}",
                 subject: "✅ SUCCESS Build",
                 body: "Pipeline OK: ${env.BUILD_URL}"
        }

        failure {
            mail to: "${env.FINAL_EMAIL}",
                 subject: "🚨 FAILED Build",
                 body: "Check logs: ${env.BUILD_URL}"
        }
    }
}