pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-cred'
        DOCKER_IMAGE          = 'muhammedsufiyan/maker'
        SONAR_SERVER_NAME     = 'sonarqube-env'
        SONAR_SCANNER_TOOL    = 'sonar-scanner'
        GIT_CREDENTIALS_ID    = 'github-cred'
        MANIFEST_REPO         = 'https://github.com/suficapz/maker.git'
        MANIFEST_BRANCH       = 'main'
    }

    stages {

        stage('Checkout Source Code') {
            steps {
                echo "Source code checked out successfully"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool "${SONAR_SCANNER_TOOL}"

                    withSonarQubeEnv("${SONAR_SERVER_NAME}") {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=maker-key \
                            -Dsonar.projectName=maker-project \
                            -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: "${DOCKER_CREDENTIALS_ID}",
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        sh """
                            echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin

                            docker build \
                            -t ${DOCKER_IMAGE}:${env.BUILD_NUMBER} \
                            -t ${DOCKER_IMAGE}:latest \
                            -f Dockerfile .

                            docker push ${DOCKER_IMAGE}:${env.BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Update Manifest Repo') {
            steps {
                dir('manifest-workspace') {

                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${MANIFEST_BRANCH}"]],
                        userRemoteConfigs: [[
                            credentialsId: "${GIT_CREDENTIALS_ID}",
                            url: "${MANIFEST_REPO}"
                        ]]
                    ])

                    withCredentials([
                        usernamePassword(
                            credentialsId: "${GIT_CREDENTIALS_ID}",
                            usernameVariable: 'GIT_USERNAME',
                            passwordVariable: 'GIT_TOKEN'
                        )
                    ]) {
                        sh """
                            sed -i 's|image: IMAGE_PLACEHOLDER|image: ${DOCKER_IMAGE}:${env.BUILD_NUMBER}|' manifests/deployment.yaml

                            git config user.name "suficapz"
                            git config user.email "sufiyanathikal@gamil.com"

                            git add manifests/deployment.yaml

                            if ! git diff --cached --quiet; then
                                git commit -m "Update maker image to ${env.BUILD_NUMBER}"

                                git push https://\${GIT_USERNAME}:\${GIT_TOKEN}@github.com/suficapz/maker.git HEAD:${MANIFEST_BRANCH}
                            else
                                echo "No manifest changes to commit"
                            fi
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully! Manifest repo updated with new image tag.'
        }

        failure {
            echo 'Pipeline failed. Please review the stage logs above.'
        }
    }
}
