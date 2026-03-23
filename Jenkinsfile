pipeline {
    agent any

    environment {
        SONAR_SCANNER_HOME = tool 'SonarScanner'
        IMAGE_NAME = "localhost:5000/devops-lab-app"
        IMAGE_TAG = "v${BUILD_NUMBER}"
        GITOPS_REPO = "https://github.com/VAISAALI18/devops-argocd-lab"
        GIT_USER = "VAISAALI18"
    }

    stages {

        stage('Checkout Source') {
            steps {
                echo '=== Stage 1: Checking out source code ==='
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo '=== Stage 2: Running SonarQube Analysis ==='
                withSonarQubeEnv('My Sonar Server') {
                    sh """
                        ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=devops-lab-app \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=node_modules/**
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo '=== Stage 3: Checking Quality Gate ==='
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo '=== Stage 4: Building Docker Image ==='
                sh """
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                    echo "Image built: ${IMAGE_NAME}:${IMAGE_TAG}"
                """
            }
        }

        stage('Push to Registry') {
            steps {
                echo '=== Stage 5: Pushing to Local Registry ==='
                sh """
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest
                    echo "Image pushed successfully"
                """
            }
        }

        stage('Update GitOps Repo') {
            steps {
                echo '=== Stage 6: Updating GitOps Repository ==='
                withCredentials([usernamePassword(
                    credentialsId: 'github-creds',
                    usernameVariable: 'GIT_USERNAME',
                    passwordVariable: 'GIT_PASSWORD'
                )]) {
                    sh """
                        # Clone GitOps repo
                        rm -rf /tmp/gitops-repo
                        git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/VAISAALI18/devops-argocd-lab /tmp/gitops-repo

                        # Update image tag in deployment.yaml
                        cd /tmp/gitops-repo
                        sed -i 's|image: localhost:5000/devops-lab-app:.*|image: localhost:5000/devops-lab-app:${IMAGE_TAG}|g' deployment.yaml

                        # Verify the change
                        grep image deployment.yaml

                        # Commit and push
                        git config user.email "ci@jenkins.com"
                        git config user.name "Jenkins CI"
                        git add deployment.yaml
                        git commit -m "ci: Update image tag to ${IMAGE_TAG} [skip ci]"
                        git push origin main
                    """
                }
            }
        }
    }

    post {
        success {
            echo '=== PIPELINE SUCCEEDED ==='
            echo 'Code passed Quality Gate and new image deployed to GitOps repo'
            echo 'Argo CD will now sync the new image to Kubernetes'
        }
        failure {
            echo '=== PIPELINE FAILED ==='
            echo 'Check SonarQube Quality Gate or Docker build errors'
        }
    }
}
