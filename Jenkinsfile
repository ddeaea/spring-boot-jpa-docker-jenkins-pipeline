pipeline {
    agent any

    tools {
        maven 'M2_HOME'
        jdk 'JAVA_HOME'
    }

    environment {
        SMTP_CREDS = credentials('smtp-token') // Jenkins credential for email
    }

    stages {
        stage('Clean Workspace') {
            steps {
                deleteDir()
            }
        }

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/oktadev/spring-boot-docker-example.git'
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean verify'
            }
        }

        stage('SAST - SonarQube') {
            steps {
                script {
                    def sonarSuccess = true
                    try {
                        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                            sh '''
                                mvn sonar:sonar \
                                -Dsonar.host.url=http://localhost:9000 \
                                -Dsonar.token=$SONAR_TOKEN \
                                -X
                            '''
                        }
                    } catch (err) {
                        sonarSuccess = false
                        echo "⚠️ SonarQube stage failed, but pipeline continues: ${err}"
                    }
                    if (!sonarSuccess) {
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }

        stage('SCA - Dependency Check') {
            steps {
                sh 'mvn org.owasp:dependency-check-maven:check || true'
            }
        }

        stage('Gitleaks Scan') {
            steps {
                sh '''
                    docker run --rm -v $WORKSPACE:/src \
                    zricethezav/gitleaks:latest detect --source /src --exit-code 0 || true
                '''
            }
        }

        stage('DAST - Web Scan') {
            steps {
                script {
                    // Fail-safe and authenticated ZAP scan
                    def dastSuccess = true
                    try {
                        withCredentials([usernamePassword(credentialsId: 'zap-credentials', usernameVariable: 'ZAP_USER', passwordVariable: 'ZAP_PASS')]) {
                            sh '''
                                mkdir -p zap-reports
                                chmod 777 zap-reports
                                docker run --rm -t \
                                -v $(pwd)/zap-reports:/zap/wrk \
                                ghcr.io/zaproxy/zaproxy:stable \
                                zap-baseline.py \
                                -t http://192.168.33.10:8080 \
                                -r zap_report.html \
                                -J zap_out.json \
                                -u $ZAP_USER \
                                -p $ZAP_PASS \
                                -I -d || true
                            '''
                        }
                    } catch (err) {
                        dastSuccess = false
                        echo "⚠️ DAST scan encountered warnings/errors, but pipeline continues: ${err}"
                    }
                    if (!dastSuccess) {
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Déploiement de l’application Spring Boot...'
                sh '''
                    nohup java -jar target/demo-0.0.1-SNAPSHOT.jar > app.log 2>&1 &
                    echo "Application Spring Boot démarrée sur le serveur Jenkins"
                '''
            }
        }
    }

    post {
        always {
            echo 'Pipeline terminé.'
        }

        success {
            mail to: 'khalilsoltani64@gmail.com',
                 subject: "✅ SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """Bonjour,

Le pipeline du projet *${env.JOB_NAME}* s'est exécuté avec succès ✅

🔗 Détails du build : ${env.BUILD_URL}

Cordialement,
Le serveur Jenkins"""
        }

        unstable {
            mail to: 'khalilsoltani64@gmail.com',
                 subject: "⚠️ UNSTABLE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """Bonjour,

Le pipeline du projet *${env.JOB_NAME}* a terminé avec des avertissements ❗

🔗 Consultez les logs ici : ${env.BUILD_URL}

Cordialement,
Le serveur Jenkins"""
        }

        failure {
            mail to: 'khalilsoltani64@gmail.com',
                 subject: "❌ ÉCHEC: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """Bonjour,

Le pipeline du projet *${env.JOB_NAME}* a échoué ❗

🔗 Consultez les logs ici : ${env.BUILD_URL}

Cordialement,
Le serveur Jenkins"""
        }
    }
}
