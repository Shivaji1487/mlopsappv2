pipeline {
    agent any

    environment {
        DOCKER_HUB_USER        = "sshivaji555"
        IMAGE_NAME             = "mlops-app"
        IMAGE_TAG              = "v1.0.${BUILD_NUMBER}"

        // MLflow & MinIO
        MINIO_ENDPOINT         = "http://192.168.235.130:9000"
        MLFLOW_S3_ENDPOINT_URL = "http://192.168.235.130:9000"
        MLFLOW_TRACKING_URI    = "http://192.168.235.130:5000"
        AWS_DEFAULT_REGION     = "us-east-1"

        // Jenkins Credentials (Username/Password)
        MINIO_CREDS            = credentials('s3credentials')
        AWS_ACCESS_KEY_ID      = "${MINIO_CREDS_USR}"
        AWS_SECRET_ACCESS_KEY  = "${MINIO_CREDS_PSW}"
    }

    stages {

        stage('Stage 1: Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Stage 2: Setup Python Environment') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Stage 3: Run Unit Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    python -m pytest test/
                '''
            }
        }

        stage('Stage 4: Validate Data') {
            steps {
                sh '''
                    . venv/bin/activate
                    python -m src.validate
                '''
            }
        }

        stage('Stage 5: Train Model') {
            steps {
                sh '''
                    . venv/bin/activate
                    python -m src.train
                '''
            }
        }

        stage('Stage 6: Build Docker Image') {
            steps {
                sh """
                    docker build \
                    -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} \
                    -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest .
                """
            }
        }

        stage('Stage 7: Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh """
                        echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest
                        docker logout
                    """
                }
            }
        }

        stage('Stage 8: Update GitOps Repository') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'gitops-token',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh """
                        rm -rf mlops-gitospv2
                        git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/Shivaji1487/mlops-gitospv2.git
                        cd mlops-gitospv2

                        git config user.name "Jenkins CI"
                        git config user.email "jenkins@local"

                        sed -i "s|repository: .*|repository: ${DOCKER_HUB_USER}/${IMAGE_NAME}|g" chart/ml-service/values-dev.yaml
                        sed -i "s|tag: .*|tag: '${IMAGE_TAG}'|g" chart/ml-service/values-dev.yaml

                        git add chart/ml-service/values-dev.yaml
                        git commit -m "Deploy Image ${IMAGE_TAG}" || true
                        git remote set-url origin https://${GIT_USER}:${GIT_TOKEN}@github.com/Shivaji1487/mlops-gitospv2.git
                        git push origin main
                    """
                }
            }
        }
    }

    post {
        always {
            sh """
                docker rmi ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} || true
                docker rmi ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest || true
            """
            cleanWs()
        }
    }
}