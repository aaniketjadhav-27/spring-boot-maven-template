pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'SonarQube-Scanner'
        APP_NAME     = 'devsecops-app'
        NEXUS_IP     = 'localhost'
        AWS_REGION   = 'ap-south-1'            // Your region
        AWS_ACCOUNT  = '966137697484'          // Your 12-digit AWS Account ID
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        // 1. Compile & package first so ~/.m2 has all dependencies locally
        stage('Maven Build & Package') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        // 2. Scan offline using the local filesystem & cached jars (avoids HTTP 429)
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --offline-scan --skip-version-check --severity HIGH,CRITICAL --format table -o trivy-fs.txt .'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=${APP_NAME} \
                    -Dsonar.sources=. \
                    -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'N_USER', passwordVariable: 'N_PASS')]) {
                    sh '''
                    JAR_FILE=$(ls target/*.jar | head -n 1)
                    curl -v -u "$N_USER:$N_PASS" \
                         --upload-file "$JAR_FILE" \
                         "http://${NEXUS_IP}:8081/repository/maven-releases/com/devsecops/${APP_NAME}/${BUILD_NUMBER}/${APP_NAME}-${BUILD_NUMBER}.jar"
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${APP_NAME}:${BUILD_NUMBER} ."
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --skip-version-check --severity CRITICAL --exit-code 0 ${APP_NAME}:${BUILD_NUMBER}"
            }
        }

        stage('Push to Amazon ECR') {
            steps {
                sh """
                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com
                docker tag ${APP_NAME}:${BUILD_NUMBER} ${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}:${BUILD_NUMBER}
                docker tag ${APP_NAME}:${BUILD_NUMBER} ${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}:latest
                docker push ${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}:${BUILD_NUMBER}
                docker push ${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}:latest
                """
            }
        }

        stage('Deploy Container to EC2') {
            steps {
                sh """
                docker stop live-devsecops-app || true
                docker rm live-devsecops-app || true
                docker run -d --name live-devsecops-app -p 8082:8080 ${APP_NAME}:${BUILD_NUMBER}
                """
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-fs.txt', allowEmptyArchive: true
        }
    }
}
