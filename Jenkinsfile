pipeline {
    agent any

    tools {
        maven 'M2_HOME'
        jdk 'JAVA_HOME'
    }

    environment {
        SONAR_TOKEN = credentials('sonar-token-id')
    }

    stages {
        stage('Build & Test') {
            steps {
                sh 'mvn clean install'
            }
        }

        stage('Dependency Check') {
            steps {
                sh 'mvn org.owasp:dependency-check-maven:check'
                archiveArtifacts artifacts: 'target/dependency-check-report.html', allowEmptyArchive: true
            }
        }

        stage('Gitleaks Scan') {
            steps {
                sh '''
                docker run --rm -v ${WORKSPACE}:/src zricethezav/gitleaks:latest detect --source /src --exit-code 0
                '''
            }
        }

        stage('Deploy') {
            steps {
                echo "Déploiement de l’application Spring Boot..."
                sh 'nohup java -jar target/demo-0.0.1-SNAPSHOT.jar &'
            }
        }
    }

    post {
        always {
            echo 'Pipeline terminé.'
        }
        success {
            mail to: 'you@example.com',
                 subject: "Pipeline Success",
                 body: "La build Jenkins a réussi."
        }
        unstable {
            mail to: 'you@example.com',
                 subject: "Pipeline Unstable",
                 body: "La build Jenkins est instable. Vérifiez les rapports de sécurité."
        }
        failure {
            mail to: 'you@example.com',
                 subject: "Pipeline Failed",
                 body: "La build Jenkins a échoué."
        }
    }
}
