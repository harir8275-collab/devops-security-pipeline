pipeline {
    agent any

    environment {
        IMAGE_NAME     = "devops-pipeline-demo"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        CONTAINER_NAME = "pipeline-demo-test"
        S3_BUCKET      = "devops-pipeline-demo-reports-hari"
        REPORT_DIR     = "reports"
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
                    mkdir -p $REPORT_DIR

                    # Human-readable table report
                    trivy image --severity HIGH,CRITICAL --format table \
                      -o $REPORT_DIR/trivy-report-$BUILD_NUMBER.txt $IMAGE_NAME:$IMAGE_TAG || true

                    # Machine-readable JSON report (useful for later automation/dashboards)
                    trivy image --severity HIGH,CRITICAL --format json \
                      -o $REPORT_DIR/trivy-report-$BUILD_NUMBER.json $IMAGE_NAME:$IMAGE_TAG || true

                    cat $REPORT_DIR/trivy-report-$BUILD_NUMBER.txt
                '''
            }
        }

        stage('Security Gate') {
            steps {
                script {
                    def criticalCount = sh(
                        script: "grep -c CRITICAL ${REPORT_DIR}/trivy-report-${BUILD_NUMBER}.txt || true",
                        returnStdout: true
                    ).trim()
                    echo "Critical vulnerabilities found: ${criticalCount}"
                    if (criticalCount.toInteger() > 0) {
                        echo "WARNING: Critical vulnerabilities detected. Review the report."
                        // Uncomment to hard-fail the build instead of just warning:
                        // error("Build failed due to ${criticalCount} critical vulnerabilities.")
                    }
                }
            }
        }

        stage('Upload Reports to S3') {
            steps {
                sh '''
                    aws s3 cp $REPORT_DIR/trivy-report-$BUILD_NUMBER.txt \
                      s3://$S3_BUCKET/trivy-reports/build-$BUILD_NUMBER/trivy-report-$BUILD_NUMBER.txt

                    aws s3 cp $REPORT_DIR/trivy-report-$BUILD_NUMBER.json \
                      s3://$S3_BUCKET/trivy-reports/build-$BUILD_NUMBER/trivy-report-$BUILD_NUMBER.json
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
            archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
        }
        success {
            echo "Pipeline completed successfully — build #${BUILD_NUMBER}. Reports uploaded to s3://${S3_BUCKET}/trivy-reports/build-${BUILD_NUMBER}/"
        }
        failure {
            echo "Pipeline failed — check logs above."
        }
    }
}
       
