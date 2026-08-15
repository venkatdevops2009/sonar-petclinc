pipeline {
    agent {
        node {
            label 'roboshop'
        }
    }

    environment {
        ACC_ID      = "843916760700"
        REGION      = "us-east-1"

        APP_REPO    = "petclinic"
        MYSQL_REPO  = "petclinic-mysql"

        IMAGE_TAG   = "${BUILD_NUMBER}"
    }

    options {
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh '''
                    mvn clean verify
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=petclinic \
                        -Dsonar.projectName=petclinic
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Verify Artifact') {
            steps {
                sh '''
                    ls -ltr target/
                '''
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh '''
                    trivy fs \
                    --scanners vuln,secret,misconfig \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    .
                '''
            }
        }

        stage('Docker Compose Build') {
            steps {
                sh '''
                    docker compose build

                    docker images | grep petclinc
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 1 \
                    piridi/petclinc:v1

                    trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 1 \
                    piridi/petclinc-mysql:v1
                '''
            }
        }

        stage('ECR Login') {
            steps {
                withAWS(credentials: 'aws-creds', region: "${REGION}") {
                    sh '''
                        aws ecr get-login-password --region ${REGION} | \
                        docker login --username AWS --password-stdin \
                        ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com
                    '''
                }
            }
        }

        stage('Tag Images') {
            steps {
                sh '''
                    docker tag piridi/petclinc:v1 \
                    ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${APP_REPO}:${IMAGE_TAG}

                    docker tag piridi/petclinc-mysql:v1 \
                    ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${MYSQL_REPO}:${IMAGE_TAG}
                '''
            }
        }

        stage('Push Images') {
            steps {
                sh '''
                    docker push \
                    ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${APP_REPO}:${IMAGE_TAG}

                    docker push \
                    ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${MYSQL_REPO}:${IMAGE_TAG}
                '''
            }
        }
    }

    post {

        success {
            echo 'Build, SonarQube, Trivy and ECR Push Successful'
        }

        failure {
            echo 'Pipeline Failed'
        }

        always {
            sh '''
                docker image prune -f || true
            '''
            cleanWs()
        }
    }
}