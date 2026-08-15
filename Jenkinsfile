pipeline {
    agent {
        node {
            label 'roboshop'
        }
    }

    environment {
        ACC_ID    = "843916760700"
        REGION    = "us-east-1"
        PROJECT   = "petclinic"

        VERSION   = "1.0.0"
        IMAGE_TAG = "${BUILD_NUMBER}"

        ECR_REPO  = "${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}"
    }

    options {
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {

        stage('Build & Unit Tests') {
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

        stage('Trivy FileSystem Scan') {
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

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t ${PROJECT}:${IMAGE_TAG} .

                    docker tag ${PROJECT}:${IMAGE_TAG} \
                    ${ECR_REPO}:${IMAGE_TAG}
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 1 \
                    ${ECR_REPO}:${IMAGE_TAG}
                '''
            }
        }

        stage('Push To ECR') {
            steps {
                withAWS(credentials: 'aws-creds', region: "${REGION}") {

                    sh '''
                        aws ecr get-login-password --region ${REGION} | \
                        docker login --username AWS --password-stdin \
                        ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com

                        docker push ${ECR_REPO}:${IMAGE_TAG}
                    '''
                }
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
            cleanWs()
        }
    }
}
