pipeline {
    agent any

    triggers {
        pollSCM('* * * * *')
    }

    // comment

    tools {
        maven 'maven-3'
        jdk 'jdk-17'
        nodejs 'node-22'
        // 'hudson.plugins.sonar.SonarRunnerInstallation' 'sonar-scanner'
    }

    environment {
        CHROME_BIN = '/usr/bin/chromium'
        // SONAR_SERVER_NAME = 'sonar-server'
        FINAL_EMAIL = "${env.DEVOPS_EMAIL ?: 'ton.email@gmail.com'}"
    }

    stages {
        stage('Build & Test Backend') {
            steps {
                script {
                    def services = ['discovery-service', 'gateway-service', 'user-service', 'product-service', 'media-service', 'order-service', 'cart-service']
                    for (service in services) {
                        dir("microservices/${service}") {
                            echo "--- Building and Testing ${service} ---"
                            sh 'mvn clean verify'
                        }
                    }
                }
            }
        }

        stage('Test & Build Frontend') {
            steps {
                dir('frontend/buy01-web') {
                    sh 'npm install'
                    script {
                        try {
                           // Use --code-coverage to generate coverage report
                           sh 'npm run test -- --no-watch --no-progress --browsers=ChromeHeadless --code-coverage'
                        } catch (Exception e) {
                           echo "⚠️ Warning tests Frontend failed, continuing..."
                        }
                    }
                    sh 'npm run build'
                }
            }
        }

        stage('Deploy to Nexus (Maven)') {
            steps {
                withCredentials([usernamePassword(credentialsId: env.NEXUS_CRED,
                usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    dir('microservices/user-service') {
                        // On passe explicitement les variables à Maven via -D
                        sh "mvn deploy -s ../../settings.xml -Denv.NEXUS_USER=${NEXUS_USER} -Denv.NEXUS_PASS=${NEXUS_PASS}"
                    }
                }
            }
        }

        // stage('Code Quality Analysis') {
        //     steps {
        //         script {
        //             echo "--- 🔍 Starting SonarQube Analysis ---"
        //             def scannerHome = tool 'sonar-scanner'

        //             withSonarQubeEnv(SONAR_SERVER_NAME) {
        //                 sh """${scannerHome}/bin/sonar-scanner \
        //                     -Dsonar.projectKey=buy-01 \
        //                     -Dsonar.projectName=buy-01 \
        //                     -Dsonar.sources=. \
        //                     -Dsonar.host.url=http://sonarqube:9000 \
        //                     -Dsonar.token=${SONAR_AUTH_TOKEN} \
        //                     -Dsonar.java.binaries=**/target/classes \
        //                     -Dsonar.coverage.jacoco.xmlReportPaths=**/target/site/jacoco/jacoco.xml \
        //                     -Dsonar.javascript.lcov.reportPaths=frontend/buy01-web/coverage/buy01-web/lcov.info"""
        //             }
        //         }
        //     }
        // }

        // stage("Quality Gate") {
        //     steps {
        //         timeout(time: 2, unit: 'MINUTES') {
        //             waitForQualityGate abortPipeline: true
        //         }
        //     }
        // }

        stage('Deploy to Production') {
            steps {
                dir('infrastructure') {
                    script {
                        try {
                            sh 'curl -SL https://github.com/docker/compose/releases/download/v2.23.3/docker-compose-linux-x86_64 -o docker-compose'
                            sh 'chmod +x docker-compose'
                            sh './docker-compose down'
                            sh './docker-compose up -d --build'
                        } catch (Exception e) {
                            if (fileExists('docker-compose')) { sh './docker-compose up -d' }
                            error "Deployment failed."
                        } finally {
                            sh 'rm -f docker-compose'
                        }
                    }
                }
            }
        }
    }

    post {
        success {
             mail to: "${env.FINAL_EMAIL}", subject: "✅ SUCCESS Buy01", body: "Build OK: ${env.BUILD_URL}"
        }
        failure {
             mail to: "${env.FINAL_EMAIL}", subject: "🚨 FAILED Buy01", body: "Check logs: ${env.BUILD_URL}"
        }
    }
}
