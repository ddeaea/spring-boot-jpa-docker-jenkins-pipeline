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

        stage('Deploy') {
            steps {
                sh '''
                    echo "Deploiement de l'application Spring Boot avec JPA..."
                    # Arreter toute instance existante
                    pkill -f "spring-boot-jpa-docker-jenkins-pipeline" || true
                    sleep 3
                    
                    # Demarrer l'application avec le bon contexte
                    nohup java -jar target/spring-boot-jpa-docker-jenkins-pipeline-0.0.1-SNAPSHOT.jar > app.log 2>&1 &
                    
                    # Attendre le demarrage
                    sleep 30
                    
                    echo "Verification du demarrage..."
                    echo "=== LOGS APPLICATION ==="
                    tail -15 app.log
                    
                    # Tester l'application sur l'IP réseau pour ZAP
                    if curl -s --connect-timeout 15 http://192.168.33.10:8080/spring-boot-jenkins/hello > /dev/null; then
                        echo "✅ APPLICATION DEMARREE ET ACCESSIBLE SUR LE RESEAU"
                        echo "🌐 URL pour ZAP: http://192.168.33.10:8080/spring-boot-jenkins"
                    else
                        echo "❌ Application non accessible sur l'IP réseau"
                    fi
                '''
            }
        }

        stage('DAST - Web Scan') {
            steps {
                script {
                    sh '''
                        echo "Création du dossier pour les rapports ZAP..."
                        mkdir -p $WORKSPACE/zap-reports
                        
                        echo "Lancement du scan DAST ZAP..."
                        
                        # Scan ZAP avec volume monté sur le workspace
                        docker run --rm \
                            -v $WORKSPACE/zap-reports:/zap/wrk:rw \
                            ghcr.io/zaproxy/zaproxy:stable \
                            zap-baseline.py \
                            -t http://192.168.33.10:8080/spring-boot-jenkins \
                            -r /zap/wrk/zap_report.html \
                            -J /zap/wrk/zap_report.json \
                            -x /zap/wrk/zap_report.xml \
                            -a -I -d -T 60
                        
                        echo "=== VERIFICATION DES RAPPORTS ZAP DANS WORKSPACE ==="
                        echo "Chemin des rapports: $WORKSPACE/zap-reports/"
                        ls -la $WORKSPACE/zap-reports/
                        echo "=== CONTENU DU WORKSPACE ==="
                        find $WORKSPACE -name ".html" -o -name ".json" -o -name "*.xml" | grep -v node_modules
                    '''
                }
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
