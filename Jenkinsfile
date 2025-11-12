pipeline {
    agent any

    tools {
        maven 'M2_HOME'
        jdk 'JAVA_HOME'
    }

    environment {
        SONAR_TOKEN = credentials('sonar-token') // Token SonarQube
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
                sh "mvn sonar:sonar -Dsonar.host.url=http://localhost:9000 -Dsonar.login=${SONAR_TOKEN}"
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

}
