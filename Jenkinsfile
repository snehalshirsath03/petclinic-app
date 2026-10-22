cd ~/petclinic-app

rm -f .Jenkinsfile.swp

cat > Jenkinsfile <<'EOF'
pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-southeast-2'
        AWS_ACCOUNT_ID = '401106839558'
        ECR_REPOSITORY = 'petclinic-app'
        IMAGE_TAG = 'latest'
        APP_PORT = '18081'
        NEXUS_REGISTRY = 'localhost:8083'
        NEXUS_IMAGE = 'localhost:8083/petclinic-app'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Unit Tests') {
            steps {
                sh '''
                    set -e
                    java -version
                    mvn -version
                    mvn clean verify
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true,
                          testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -e
                    docker build -t petclinic-app:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Start Temporary Test App') {
            steps {
                sh '''
                    set -e
                    docker rm -f petclinic-test 2>/dev/null || true
                    docker run -d \
                      --name petclinic-test \
                      -p ${APP_PORT}:8080 \
                      petclinic-app:${BUILD_NUMBER}

                    for i in $(seq 1 30); do
                      if curl -fsS http://localhost:${APP_PORT}/ -o /dev/null; then
                        echo "Temporary PetClinic test app is ready."
                        exit 0
                      fi
                      sleep 5
                    done

                    docker logs --tail 50 petclinic-test
                    exit 1
                '''
            }
        }

        stage('Functional Test') {
            steps {
                sh '''
                    set -e
                    STATUS=$(curl -sS -o /dev/null \
                      -w "%{http_code}" \
                      http://localhost:${APP_PORT}/)
                    echo "PetClinic HTTP status: ${STATUS}"
                    test "${STATUS}" = "200"
                    echo "Functional test passed."
                '''
            }
        }

        stage('Performance Test') {
            steps {
                sh '''
                    set -e
                    command -v ab >/dev/null 2>&1 || {
                      echo "ApacheBench (ab) is not installed. Install apache2-utils first."
                      exit 1
                    }
                    ab -n 100 -c 5 http://localhost:${APP_PORT}/
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    withCredentials([
                        string(
                            credentialsId: 'sonarqube-token',
                            variable: 'SONAR_TOKEN'
                        )
                    ]) {
                        sh '''
                            set -e
                            mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.3.0.6276:sonar \
                              -Dsonar.projectKey=petclinic-app \
                              -Dsonar.host.url=http://localhost:9000 \
                              -Dsonar.token="$SONAR_TOKEN"
                        '''
                    }
                }
            }
        }

        stage('Push Image to Nexus') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-docker-creds',
                        usernameVariable: 'NEXUS_USERNAME',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -e
                        printf '%s' "$NEXUS_PASSWORD" |
                          docker login "$NEXUS_REGISTRY" \
                            --username "$NEXUS_USERNAME" \
                            --password-stdin

                        docker tag "petclinic-app:${BUILD_NUMBER}" \
                          "${NEXUS_IMAGE}:${BUILD_NUMBER}"
                        docker push "${NEXUS_IMAGE}:${BUILD_NUMBER}"

                        docker tag "petclinic-app:${BUILD_NUMBER}" \
                          "${NEXUS_IMAGE}:latest"
                        docker push "${NEXUS_IMAGE}:latest"
                    '''
                }
            }
        }

        stage('Push Image to Amazon ECR') {
            steps {
                sh '''
                    set -e
                    ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"

                    aws ecr get-login-password --region "${AWS_REGION}" |
                      docker login --username AWS --password-stdin "${ECR_REGISTRY}"

                    docker tag "petclinic-app:${BUILD_NUMBER}" "${ECR_IMAGE}"
                    docker push "${ECR_IMAGE}"
                '''
            }
        }
    }

    post {
        always {
            sh '''
                docker rm -f petclinic-test 2>/dev/null || true
            '''
            echo 'Temporary test container cleanup completed.'
        }
    }
}
EOF
