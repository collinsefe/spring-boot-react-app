pipeline {
    agent any

    environment {
        AWS_CREDENTIALS = 'aws-credentials'
        EC2_USER = 'ec2-user'
        EC2_HOST = '35.176.196.120'
        EC2_KEY = 'ssh-ec2-user' 
        APP_DIR = 'spring-boot-app'
        PORT = 8082
    }

    stages {
        stage('Build Application') {
            steps {
                echo 'Building the Maven application...'
                sh 'mvn clean install'
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    // Archive the built JAR file and transfer to EC2
                    sshagent(credentials: [EC2_KEY]) {
                        sh """
                        scp -o StrictHostKeyChecking=no target/*.jar ${EC2_USER}@${EC2_HOST}:/home/${EC2_USER}/${APP_DIR}/app.jar
                        """
                    }
                    
                    // Connect to the EC2 instance and run the JAR file
                    sshagent(credentials: [EC2_KEY]) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} << EOF
                            pkill -f 'java -jar' || true
                            nohup java -jar /home/${EC2_USER}/${APP_DIR}/app.jar --server.port=${PORT} > /home/${EC2_USER}/${APP_DIR}/application.log 2>&1 &
                            sleep 10
                            echo 'Application deployed and started on EC2!'
                        EOF
                        """
                    }
                }
            }
        }

        stage('Test Application') {
            steps {
                echo "Testing if the application is running on EC2 instance..."
                script {
                    def response = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://${EC2_HOST}:${PORT}/api/employees/3", returnStdout: true).trim()
                    if (response == '200') {
                        echo 'Application is running and responded with HTTP 200 OK!'
                    } else {
                        error "Application test failed! Endpoint responded with HTTP ${response}"
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed.'
        }
    }
}
