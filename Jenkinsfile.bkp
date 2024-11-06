
pipeline {
    agent any
    environment {
        PORT = 8082
    }
    stages {
        stage('Clone Repository') {
            steps {
                git url: 'https://gitlab.com/cloud-devops-assignments/spring-boot-react-example.git'
            }
        }
        stage('Build Application') {
            steps {
                echo 'Building the application...'
                sh 'mvn clean install'
            }
        }
        stage('Run Application') {
            steps {
                echo 'Running the Spring Boot application...'
                sh """
                nohup java -jar target/*.jar --server.port=${PORT} > application.log 2>&1 &
                echo "Application started in detached mode with PID: \$!"
                """
            }
        }
        stage('Test API') {
            steps {
                echo 'Testing the API...'
                sh """
                sleep 10
                curl -v -u greg:turnquist http://localhost:${PORT}/api/employees/3
                """
            }
        }
        
        stage('Deploy App') {
            steps {
                echo 'Testing the API...'
                // add step to deploy app
            }
        }
    }
    post {
        always {
            echo "Jenkins pipeline completed. The application is running in the background."
        }
    }
}