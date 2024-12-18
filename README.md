pipeline {
    agent any

    environment {
        // Define AWS region and credentials (use Jenkins credentials plugin)
        AWS_REGION = 'us-east-1'
        AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')  // Replace with your Jenkins AWS credentials ID
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')  // Replace with your Jenkins AWS credentials ID
        S3_BUCKET_NAME = 'my-app-artifacts-bucket'  // Replace with your S3 bucket name
        EC2_INSTANCE_IP = 'xx.xx.xx.xx'  // Replace with your EC2 instance's public IP
        EC2_USER = 'ec2-user'  // Default user for Amazon Linux AMI
        EC2_KEY_PATH = '/path/to/private-key.pem'  // Replace with the path to your private key for EC2 SSH access
    }

    stages {
        stage('Checkout Code') {
            steps {
                // Checkout the code from the Git repository
                checkout scm
            }
        }

        stage('Build') {
            steps {
                script {
                    // Build the project using Maven (you can change this based on your project)
                    echo "Building the application..."
                    sh 'mvn clean install'  // Maven build command
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    echo "Deploying application to EC2..."

                    // SCP to copy the built artifact to the EC2 instance
                    sh """
                        scp -i ${EC2_KEY_PATH} target/my-app.jar ${EC2_USER}@${EC2_INSTANCE_IP}:/home/${EC2_USER}/app/
                    """

                    // SSH into EC2 and start the application
                    sh """
                        ssh -i ${EC2_KEY_PATH} ${EC2_USER}@${EC2_INSTANCE_IP} 'java -jar /home/${EC2_USER}/app/my-app.jar &'
                    """
                }
            }
        }

        stage('Upload Artifact to S3') {
            steps {
                script {
                    echo "Uploading artifact to S3..."

                    // Upload the artifact to S3 bucket
                    sh """
                        aws s3 cp target/my-app.jar s3://${S3_BUCKET_NAME}/my-app-${BUILD_NUMBER}.jar --region ${AWS_REGION}
                    """
                }
            }
        }

        stage('Cleanup') {
            steps {
                echo "Cleaning up..."
                // Remove build files to clean up the workspace
                sh 'rm -rf target/'
            }
        }
    }

    post {
        success {
            echo "Build, deploy, and artifact upload were successful!"
        }
        failure {
            echo "There was a failure in the pipeline."
        }
    }
}
