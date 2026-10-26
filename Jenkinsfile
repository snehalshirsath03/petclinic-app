pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 45, unit: 'MINUTES')
    }

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

                    echo "=== Memory Before Build ==="
                    free -h
                '''
            }
        }

        stage('Build and Unit Tests') {
            steps {
                sh '''
                    set -eu

                    echo "=== Maven Build and Unit Tests ==="

                    ./mvnw clean package \
                        -Dtest='!PostgresIntegrationTests,!MySqlIntegrationTests'

                    echo "=== Maven Build Completed ==="
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

                    echo "=== Starting Temporary PetClinic ==="

                    docker rm -f "$TEST_CONTAINER" 2>/dev/null || true

                    docker run -d \
                        --name "$TEST_CONTAINER" \
                        -p "${APP_PORT}:8080" \
                        petclinic-app:${BUILD_NUMBER}

                    APP_READY=false

                    for i in $(seq 1 40); do
                        if curl -fsS \
                            "http://localhost:${APP_PORT}/" \
                            >/dev/null; then

                            APP_READY=true
                            echo "PetClinic is ready."
                            break
                        fi

                        echo "Waiting for PetClinic..."
                        sleep 5
                    done

                    if [ "$APP_READY" != "true" ]; then
                        echo "PetClinic failed to start."
                        docker logs "$TEST_CONTAINER"
                        exit 1
                    fi
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

                    echo "HTTP status: $HTTP_CODE"

                    test "$HTTP_CODE" = "200"

                    grep -qi "petclinic" \
                        /tmp/petclinic-response.html

                    echo "Functional test passed."
                '''
            }
        }

        stage('Performance Test') {
            steps {
                sh '''
                    set -eu

                    echo "=== Performance Test ==="

                    ab \
                        -n 100 \
                        -c 5 \
                        "http://localhost:${APP_PORT}/"
                '''
            }
        }

        stage('Stop Temporary Test App') {
            steps {
                sh '''
                    echo "=== Stop Temporary PetClinic ==="

                    docker rm -f "$TEST_CONTAINER" \
                        2>/dev/null || true

                    free -h
                '''
            }
        }

        stage('Start SonarQube and Nexus') {
            steps {
                sh '''
                    set -eu

                    echo "=== Starting SonarQube and Nexus ==="

                    docker start sonarqube
                    docker start nexus

                    echo "Waiting for SonarQube..."

                    SONAR_READY=false

                    for i in $(seq 1 90); do
                        if curl -fsS \
                            http://localhost:9000/api/system/status \
                            | grep -q '"status":"UP"'; then

                            SONAR_READY=true
                            echo "SonarQube is UP."
                            break
                        fi

                        sleep 5
                    done

                    if [ "$SONAR_READY" != "true" ]; then
                        echo "SonarQube failed to become ready."
                        docker logs sonarqube --tail 100
                        exit 1
                    fi

                    echo "Waiting for Nexus..."

                    NEXUS_READY=false

                    for i in $(seq 1 90); do
                        if curl -fsS \
                            http://localhost:8081/ \
                            >/dev/null; then

                            NEXUS_READY=true
                            echo "Nexus is ready."
                            break
                        fi

                        sleep 5
                    done

                    if [ "$NEXUS_READY" != "true" ]; then
                        echo "Nexus failed to become ready."
                        docker logs nexus --tail 100
                        exit 1
                    fi

                    echo "=== CI Services Ready ==="

                    free -h
                    docker stats --no-stream
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

                        echo "=== Push Image to Nexus ==="

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

        stage('Stop SonarQube and Nexus') {
            steps {
                sh '''
                    echo "=== Stopping SonarQube and Nexus ==="

                    docker stop sonarqube nexus || true

                    free -h
                '''
            }
        }

        stage('Push Image to Amazon ECR') {
            steps {
                sh '''
                    set -eu

                    echo "=== Push Image to Amazon ECR ==="

                    ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    ECR_URI="${ECR_REGISTRY}/${ECR_REPOSITORY}"

                    /usr/local/bin/aws sts \
                        get-caller-identity

                    /usr/local/bin/aws ecr \
                        describe-repositories \
                        --repository-names "$ECR_REPOSITORY" \
                        --region "$AWS_REGION" \
                        >/dev/null

                    /usr/local/bin/aws ecr \
                        get-login-password \
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

                    docker logout \
                        "$ECR_REGISTRY"

                    echo "ECR push completed."
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

                        echo "=== Update GitOps Manifest ==="

                        MANIFEST="argocd-manifests/deployment.yaml"

                        ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                        ECR_URI="${ECR_REGISTRY}/${ECR_REPOSITORY}"

                        test -f "$MANIFEST"

                        sed -i -E \
                            "s|image: .*petclinic-app:[^[:space:]]+|image: ${ECR_URI}:${BUILD_NUMBER}|" \
                            "$MANIFEST"

                        echo "Updated image:"

                        grep "image:" "$MANIFEST"

                        git config \
                            user.name "Jenkins"

                        git config \
                            user.email "jenkins@localhost"

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
                echo "=== Final Cleanup ==="

                docker rm -f "$TEST_CONTAINER" \
                    2>/dev/null || true

                docker ps -aq \
                    --filter "ancestor=mysql:9.7" |
                    xargs -r docker rm -f || true

                docker ps -aq \
                    --filter "ancestor=testcontainers/ryuk:0.14.0" |
                    xargs -r docker rm -f || true

                docker stop sonarqube nexus \
                    2>/dev/null || true
            '''

            echo 'Temporary containers and CI services cleaned up.'
        }

        success {
            echo 'PetClinic CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the first failed stage.'
        }
    }
}
