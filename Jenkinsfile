pipeline {
    agent any

    environment {
        IMAGE_NAME = "devops-pipeline-demo"
        IMAGE_TAG  = "${BUILD_NUMBER}"
        CONTAINER_NAME = "pipeline-demo-test"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/harir8275-collab/devops-security-pipeline.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    trivy image --severity HIGH,CRITICAL --format table \
                      -o trivy-report.txt $IMAGE_NAME:$IMAGE_TAG || true
                    cat trivy-report.txt
                '''
            }
        }

        stage('Run Container Test') {
            steps {
                sh '''
                    docker rm -f $CONTAINER_NAME || true
                    docker run -d -p 5000:5000 --name $CONTAINER_NAME $IMAGE_NAME:$IMAGE_TAG
                    sleep 5
                    curl -f http://localhost:5000/health
                '''
            }
        }

        stage('Cleanup') {
            steps {
                sh '''
                    docker rm -f $CONTAINER_NAME || true
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true
        }
        success {
            echo "Pipeline completed successfully — build #${BUILD_NUMBER}"
        }
        failure {
            echo "Pipeline failed — check logs above."
        }
    }
}
