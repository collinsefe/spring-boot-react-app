pipeline {
    agent any

    environment {
        AWS_CREDENTIALS = 'aws-credentials'
        EC2_USER = 'ec2-user'
        EC2_HOST = '35.176.196.120'
        S3_BUCKET_NAME = 'cap-gem-artifact-bucket-06112024'
        APP_DIR = 'spring-boot-app'
        PORT = 8082
        JAR_FILE = 'app.jar'
    }

    stages {
        stage('Build Application') {
            steps {
                echo 'Building the Maven application...'
                sh "'${MAVEN_HOME}/bin/mvn' clean install"
            }
        }

        stage('Push JAR to S3') {
            steps {
                echo 'Uploading JAR file to S3...'
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDENTIALS}"]]) {
                    sh """
                    aws s3 cp target/*.jar s3://${S3_BUCKET_NAME}/${JAR_FILE} || exit 1
                    """
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    // SSH into EC2 instance and pull the JAR file from S3
                    sshagent(credentials: ['collinsefe']) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} << EOF
                            # Install AWS CLI if not installed (uncomment if needed)
                            # sudo yum install aws-cli -y
                            
                            # Download the JAR file from S3
                            aws s3 cp s3://${S3_BUCKET_NAME}/${JAR_FILE} /home/${EC2_USER}/${APP_DIR}/${JAR_FILE} || exit 1

                            # Stop any existing instance of the application
                            pkill -f 'java -jar' || true

                            # Start the new instance of the application in detached mode
                            nohup java -jar /home/${EC2_USER}/${APP_DIR}/${JAR_FILE} --server.port=${PORT} > /home/${EC2_USER}/${APP_DIR}/application.log 2>&1 &
                            
                            # Confirm the application started
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
