pipeline {
    agent any

    environment {
        AWS_CREDENTIALS = 'aws-credentials' 
        AWS_ACCOUNT_ID = '684361860346' 
        AWS_REGION = 'eu-west-2'
        ECR_REPO_NAME = 'cap-gem-app-demo'
        DOCKER_IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
        DOCKER_TAG = 'demo'
        ECS_CLUSTER_NAME = 'app-cluster-demo'
        ECS_SERVICE_NAME = 'app-service-demo'
        ECS_TASK_DEFINITION = 'app-task-family-demo'
        APP_ENDPOINT = "http://app-alb-demo-1274756266.eu-west-2.elb.amazonaws.com"
    }

    stages {
        stage('Clone Repository') {
            steps {
                git url: 'https://github.com/collinsefe/spring-boot-react-example.git'
            }
        }

        stage('Build Application') {
            steps {
                echo 'Building the Maven application...'
                sh 'mvn clean install'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh "sudo docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} . || exit 1"
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                echo 'Logging in to Amazon ECR and pushing the image...'
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDENTIALS}"]]) {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | sudo docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                }
                sh "sudo docker push ${DOCKER_IMAGE}:${DOCKER_TAG} || exit 1"
            }
        }

        stage('Register ECS Task Definition') {
            steps {
                script {
                    echo 'Registering ECS Task Definition...'
                    sh """
                    aws ecs register-task-definition \
                      --family ${ECS_TASK_DEFINITION} \
                      --network-mode awsvpc \
                      --requires-compatibilities FARGATE \
                      --cpu "256" \
                      --memory "512" \
                      --execution-role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/ecsTaskExecutionRole \
                      --container-definitions '[{
                        "name": "${ECS_TASK_DEFINITION}",
                        "image": "${DOCKER_IMAGE}:${DOCKER_TAG}",
                        "essential": true,
                        "portMappings": [{"containerPort": 80, "hostPort": 0}],
                        "logConfiguration": {
                          "logDriver": "awslogs",
                          "options": {
                            "awslogs-group": "/ecs/${ECS_SERVICE_NAME}",
                            "awslogs-region": "${AWS_REGION}",
                            "awslogs-stream-prefix": "ecs"
                          }
                        }
                      }]' || exit 1
                    """
                }
            }
        }

        stage('Update ECS Service') {
            steps {
                script {
                    echo 'Updating ECS Service...'
                    sh """
                    aws ecs update-service \
                      --cluster ${ECS_CLUSTER_NAME} \
                      --service ${ECS_SERVICE_NAME} \
                      --force-new-deployment || exit 1
                    """
                }
            }
        }
        
        stage('Test Application') {
            steps {
                echo "Testing if the application is running on ${APP_ENDPOINT}..."
                sleep(20)
                script {
                    def response = sh(script: "curl -s -o /dev/null -w '%{http_code}' ${APP_ENDPOINT}", returnStdout: true).trim()
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
            echo 'Pipeline completed. Checking ECS services...'
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDENTIALS}"]]) {
                sh "aws ecs list-services --cluster ${ECS_CLUSTER_NAME} || true"
            }
        }
    }
}
