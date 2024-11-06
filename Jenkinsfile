pipeline {
    agent any

    environment {
        AWS_CREDENTIALS = 'aws-credentials'
        EC2_USER = 'ec2-user'
        EC2_HOST = '3.10.169.33'
        EC2_KEY = '35.176.196.120'
        APP_DIR = 'spring-boot-app'
        PORT = 8081
        S3_BUCKET_NAME = 'cap-gem-artifact-bucket-06112024'
        JAR_FILE = 'app.jar'
    }

    stages {
        stage('Build Application') {
            steps {
                echo 'Building the Maven application...'
                sh "mvn clean install"
            }
        }

        stage('Prepare EC2 and Deploy JAR') {
            steps {
                script {
                    sshagent(credentials: [EC2_KEY]) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} 'mkdir -p /home/${EC2_USER}/${APP_DIR}'
                        """
                        
                        sh """
                        scp -o StrictHostKeyChecking=no target/*.jar ${EC2_USER}@${EC2_HOST}:/home/${EC2_USER}/${APP_DIR}/${JAR_FILE}
                        """
                    }
                }
            }
        }

        stage('Start Application on EC2') {
            steps {
                script {
                    sshagent(credentials: [EC2_KEY]) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} <<EOF
                            pkill -f 'java -jar' || true
                            
                            nohup java -jar /home/${EC2_USER}/${APP_DIR}/${JAR_FILE} --server.port=${PORT} > /home/${EC2_USER}/${APP_DIR}/application.log 2>&1 &
                            
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
                    def response = sh(script: "curl -v -u greg:turnquist http://${EC2_HOST}:${PORT}/api/employees/3", returnStdout: true).trim()
                    echo "Response: ${response}"
                    if (response.contains("200 OK")) {
                        echo 'Application is running and responded with HTTP 200 OK!'
                    } else {
                        error "Application test failed! Endpoint responded with ${response}"
                    }
                }
            }
        }

    

        stage('Test Application') {
            steps {
                echo "Testing if the application is running on EC2 instance..."
                script {
                    def response = ''
                    def maxRetries = 3
                    def delay = 20
        
                    for (int i = 0; i < maxRetries; i++) {
                        response = sh(script: "curl -s -o /dev/null -w -v '%{http_code}' http://localhost:${PORT}", returnStdout: true).trim()
                        
                        if (response == '200') {
                            echo 'Application is running and responded with HTTP 200 OK!'
                            break
                        } else {
                            echo "Attempt ${i+1}: Application test failed with HTTP ${response}. Retrying in ${delay} seconds..."
                            sleep(delay)
                        }
                    }
        
                    if (response != '200') {
                        error "Application test failed after ${maxRetries} attempts. Endpoint responded with HTTP ${response}"
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
