pipeline {
    agent any

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-21-openjdk-amd64'
        PATH = "/usr/lib/jvm/java-21-openjdk-amd64/bin:/usr/local/bin:/usr/bin:/bin"

        AWS_REGION = 'ap-southeast-2'
        AWS_ACCOUNT_ID = '401106839558'
        ECR_REPOSITORY = 'petclinic-app'

        NEXUS_REGISTRY = 'localhost:8082'
        NEXUS_IMAGE = 'localhost:8082/petclinic-app'

        APP_PORT = '18081'
        TEST_CONTAINER = 'petclinic-test'

        SONAR_PROJECT_KEY = 'petclinic-app'
    }

    stages {

        stage('Checkout SCM') {
            steps {
                checkout scm

                sh '''
                    set -eu

                    echo "=== Environment ==="

                    java -version
                    javac -version
                    ./mvnw -version
                    docker --version
                    /usr/local/bin/aws --version
                '''
            }
        }

        stage('Build and Unit Tests') {
            steps {
                sh '''
                    set -eu

                    echo "=== Maven Build and Tests ==="

                    ./mvnw clean package \
                        -Dtest='!PostgresIntegrationTests,!MySqlIntegrationTests'
                '''
            }

            post {
                always {
                    junit(
                        allowEmptyResults: true,
                        testResults: '**/target/surefire-reports/*.xml,**/target/failsafe-reports/*.xml'
                    )
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu

                    echo "=== Docker Build ==="

                    docker build \
                        -t petclinic-app:${BUILD_NUMBER} .

                    docker tag \
                        petclinic-app:${BUILD_NUMBER} \
                        petclinic-app:latest

                    docker images | grep petclinic
                '''
            }
        }

        stage('Start Temporary Test App') {
            steps {
                sh '''
                    set -eu

                    docker rm -f "$TEST_CONTAINER" 2>/dev/null || true

                    docker run -d \
                        --name "$TEST_CONTAINER" \
                        -p "${APP_PORT}:8080" \
                        petclinic-app:${BUILD_NUMBER}

                    echo "Waiting for PetClinic to start..."

                    for i in $(seq 1 30); do

                        if curl -fsS \
                            "http://localhost:${APP_PORT}/" \
                            >/dev/null; then

                            echo "PetClinic started successfully."
                            exit 0
                        fi

                        sleep 5
                    done

                    echo "PetClinic failed to start."

                    docker logs "$TEST_CONTAINER"

                    exit 1
                '''
            }
        }

        stage('Functional Test') {
            steps {
                sh '''
                    set -eu

                    echo "=== Functional Test ==="

                    HTTP_CODE=$(curl \
                        -sS \
                        -o /tmp/petclinic-response.html \
                        -w '%{http_code}' \
                        "http://localhost:${APP_PORT}/")

                    echo "HTTP Status = $HTTP_CODE"

                    test "$HTTP_CODE" = "200"

                    echo "Functional test PASSED."
                '''
            }
        }

        stage('Performance Test') {
            steps {
                sh '''
                    set -eu

                    echo "=== Performance Test ==="

                    command -v ab

                    ab \
                        -n 100 \
                        -c 5 \
                        "http://localhost:${APP_PORT}/"
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
                            set -eu

                            echo "=== SonarQube Analysis ==="

                            ./mvnw \
                                org.sonarsource.scanner.maven:sonar-maven-plugin:5.0.0.4389:sonar \
                                -Dsonar.projectKey="$SONAR_PROJECT_KEY" \
                                -Dsonar.host.url="$SONAR_HOST_URL" \
                                -Dsonar.token="$SONAR_TOKEN"

                            echo "SonarQube analysis completed."
                        '''
                    }
                }
            }
        }

        stage('Push Image to Nexus') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-docker-credentials',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {

                    sh '''
                        set -eu

                        echo "=== Nexus Push ==="

                        echo "$NEXUS_PASSWORD" |
                            docker login \
                            "$NEXUS_REGISTRY" \
                            --username "$NEXUS_USER" \
                            --password-stdin

                        docker tag \
                            petclinic-app:${BUILD_NUMBER} \
                            "${NEXUS_IMAGE}:${BUILD_NUMBER}"

                        docker tag \
                            petclinic-app:${BUILD_NUMBER} \
                            "${NEXUS_IMAGE}:latest"

                        docker push \
                            "${NEXUS_IMAGE}:${BUILD_NUMBER}"

                        docker push \
                            "${NEXUS_IMAGE}:latest"

                        docker logout "$NEXUS_REGISTRY"

                        echo "Nexus push completed."
                    '''
                }
            }
        }

        stage('Push Image to Amazon ECR') {
            steps {
                sh '''
                    set -eu

                    echo "=== Amazon ECR Push ==="

                    ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

                    ECR_URI="${ECR_REGISTRY}/${ECR_REPOSITORY}"

                    /usr/local/bin/aws sts get-caller-identity

                    /usr/local/bin/aws ecr describe-repositories \
                        --repository-names "$ECR_REPOSITORY" \
                        --region "$AWS_REGION" \
                        >/dev/null

                    /usr/local/bin/aws ecr get-login-password \
                        --region "$AWS_REGION" |
                        docker login \
                            --username AWS \
                            --password-stdin \
                            "$ECR_REGISTRY"

                    docker tag \
                        petclinic-app:${BUILD_NUMBER} \
                        "${ECR_URI}:${BUILD_NUMBER}"

                    docker tag \
                        petclinic-app:${BUILD_NUMBER} \
                        "${ECR_URI}:latest"

                    docker push \
                        "${ECR_URI}:${BUILD_NUMBER}"

                    docker push \
                        "${ECR_URI}:latest"

                    docker logout "$ECR_REGISTRY"

                    echo "Amazon ECR push completed."
                '''
            }
        }

        stage('Update GitOps Manifest') {
            steps {

                withCredentials([
                    string(
                        credentialsId: 'github-token',
                        variable: 'GITHUB_TOKEN'
                    )
                ]) {

                    sh '''
                        set -eu

                        echo "=== GitOps Manifest Update ==="

                        MANIFEST="argocd-manifests/deployment.yaml"

                        ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

                        ECR_URI="${ECR_REGISTRY}/${ECR_REPOSITORY}"

                        test -f "$MANIFEST"

                        sed -i -E \
                            "s|image: .*petclinic-app:[^[:space:]]+|image: ${ECR_URI}:${BUILD_NUMBER}|" \
                            "$MANIFEST"

                        echo "New deployment image:"

                        grep "image:" "$MANIFEST"

                        git config user.name "Jenkins"

                        git config user.email "jenkins@localhost"

                        git add "$MANIFEST"

                        if git diff --cached --quiet; then

                            echo "Manifest already contains this image."

                        else

                            git commit \
                                -m "Update PetClinic image to build ${BUILD_NUMBER} [skip ci]"

                            ASKPASS="$(mktemp)"

                            trap 'rm -f "$ASKPASS"' EXIT

                            cat > "$ASKPASS" <<'ASKPASS_SCRIPT'
#!/bin/sh

case "$1" in

    *Username*)
        printf '%s\n' 'x-access-token'
        ;;

    *Password*)
        printf '%s\n' "$GITHUB_TOKEN"
        ;;

    *)
        exit 1
        ;;

esac
ASKPASS_SCRIPT

                            chmod 700 "$ASKPASS"

                            GIT_ASKPASS="$ASKPASS" \
                            GIT_TERMINAL_PROMPT=0 \
                                git push origin HEAD:main
                        fi
                    '''
                }
            }
        }
    }

    post {

        always {

            sh '''
                docker rm -f "$TEST_CONTAINER" \
                    2>/dev/null || true
            '''

            echo 'Temporary PetClinic test container removed.'
        }

        success {
            echo 'PetClinic CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'PetClinic CI/CD pipeline failed. Check the first failed stage.'
        }
    }
}
