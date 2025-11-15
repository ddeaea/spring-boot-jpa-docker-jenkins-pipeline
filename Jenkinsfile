pipeline {
    agent any

    tools {
        maven 'M2_HOME'
        jdk 'JAVA_HOME'
    }

    // 👇 Add your SMTP credentials (you created this in Jenkins → Credentials)
    environment {
        SMTP_CREDS = credentials('smtp-token')  // replace with your actual credential ID
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

        stage('SAST - SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh 'mvn sonar:sonar -Dsonar.host.url=http://localhost:9000 -Dsonar.login=$SONAR_TOKEN'
                }
            }
        }

        stage('SCA - Dependency Check') {
            steps {
                sh 'mvn org.owasp:dependency-check-maven:check'
            }
        }

        stage('Gitleaks Scan') {
            steps {
                sh '''
                    docker run --rm -v $WORKSPACE:/src zricethezav/gitleaks:latest detect --source /src --exit-code 0
                '''
            }
        }

        stage('DAST - Web Scan') {
            steps {
                script {
                    sh '''
                        mkdir -p zap-reports
                        chmod 777 zap-reports
                        docker run --rm -t \
                        -v $(pwd)/zap-reports:/zap/wrk \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py -t http://192.168.33.10:8080 \
                        -r zap_report.html -J zap_out.json -I -d
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Déploiement de l’application Spring Boot...'
                sh '''
                    java -jar target/demo-0.0.1-SNAPSHOT.jar &
                    echo "Application Spring Boot démarrée sur le serveur Jenkins"
                '''
            }
        }
    }

    post {
        always {
            echo 'Pipeline terminé !'
        }

        // ✅ Email on success
        success {
            mail to: 'your.email@example.com',
                 subject: "✅ Build Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """Bonjour,

Le pipeline du projet *${env.JOB_NAME}* s'est exécuté avec succès.

➡️ Détails du build : ${env.BUILD_URL}

Cordialement,
Le serveur Jenkins""",
                 replyTo: "${env.SMTP_CREDS_USR}"
        }

        // ⚠️ Email on failure
        failure {
            mail to: 'your.email@example.com',
                 subject: "❌ Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """Bonjour,

Le pipeline du projet *${env.JOB_NAME}* a échoué à l’étape : ${env.STAGE_NAME}.

➡️ Consultez les logs : ${env.BUILD_URL}

Cordialement,
Le serveur Jenkins""",
                 replyTo: "${env.SMTP_CREDS_USR}"
        }
    }
}
