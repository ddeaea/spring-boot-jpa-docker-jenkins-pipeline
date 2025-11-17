pipeline {
    agent any

    tools {
        maven 'M2_HOME'
        jdk 'JAVA_HOME'
    }

    environment {
        SONAR_TOKEN = credentials('sonar-token')
    }

    stages {

        stage('Clean Workspace') {
            steps {
                deleteDir()
            }
        }

        stage('Checkout') {
            steps {
                git branch: 'patch-1', url: 'https://github.com/MonomNakhli/spring-boot-jpa-docker-jenkins-pipeline.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SAST - SonarQube') {
            steps {
                // Prevent pipeline from failing
                sh '''
                    set +e
                    mvn sonar:sonar \
                        -Dsonar.host.url=http://localhost:9000 \
                        -Dsonar.login=$SONAR_TOKEN
                    echo "SonarQube finished with exit code $? (ignored)"
                '''
            }
        }

        stage('SCA - Dependency Check') {
            steps {
                sh '''
                    set +e
                    mvn org.owasp:dependency-check-maven:check
                    echo "Dependency-Check finished with exit code $? (ignored)"
                '''
            }
        }

        stage('Gitleaks Scan') {
            steps {
                sh '''
                    set +e
                    docker run --rm -v $WORKSPACE:/src \
                        zricethezav/gitleaks:latest detect --source /src --exit-code 0
                    echo "Gitleaks scan done."
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    echo "Déploiement Spring Boot..."

                    pkill -f "spring-boot-jpa-docker-jenkins-pipeline" || true
                    sleep 3
                    
                    nohup java -jar target/spring-boot-jpa-docker-jenkins-pipeline-0.0.1-SNAPSHOT.jar \
                        > app.log 2>&1 &
                    
                    sleep 15
                    
                    echo "=== LAST LOGS ==="
                    tail -20 app.log || true
                '''
            }
        }

        stage('DAST - ZAP Scan') {
            steps {
                sh '''
                    set +e
                    mkdir -p $WORKSPACE/zap-reports

                    docker run --rm \
                        -v $WORKSPACE/zap-reports:/zap/wrk \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py \
                        -t http://192.168.33.10:8080/spring-boot-jenkins \
                        -r /zap/wrk/zap_report.html \
                        -J /zap/wrk/zap_report.json \
                        -x /zap/wrk/zap_report.xml \
                        -a -I -T 60

                    echo "ZAP scan completed (errors ignored)"
                '''
            }
        }
    }

    post {
        always {
            echo "Pipeline DevSecOps terminé"
            echo "=== EMPLACEMENT DES RAPPORTS ==="
            sh '''
                echo "Rapports ZAP: $WORKSPACE/zap-reports/"
                echo "Rapport Dependency Check: $WORKSPACE/target/dependency-check-report.html"
                echo "Logs application: $WORKSPACE/app.log"
            '''
        }
        success {
            emailext(
                to: "khalilsoltani64@gmail.com",
                subject: "Pipeline DevSecOps réussi : ${currentBuild.fullDisplayName}",
                body: "Le pipeline DevSecOps a été exécuté avec succès. Les rapports sont dans le workspace Jenkins."
            )
        }
        failure {
            emailext(
                to: "khalilsoltani64@gmail.com",
                subject: "Pipeline DevSecOps échoué : ${currentBuild.fullDisplayName}",
                body: "Le pipeline a échoué. Consultez les logs: ${env.BUILD_URL}"
            )
        }
    }
}

