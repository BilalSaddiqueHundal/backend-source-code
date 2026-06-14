pipeline {
    agent any

    environment {
        REGISTRY_USER = "bilalsaddique"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        GITOPS_REPO = "https://github.com/BilalSaddiqueHundal/ITC-backend-gitops.git"
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test Services') {
            steps {
                script {
                    def services = [
                        'connectionservice',
                        'apigateway'
                    ]

                    for (service in services) {
                        dir(service) {
                            sh 'mvn clean test package'
                        }
                    }
                }
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                }
            }
        }

        stage('Build and Push Docker Images') {
            steps {
                script {
                    def services = [
                        'connectionservice',
                        'apigateway'
                    ]

                    for (service in services) {
                        dir(service) {
                            sh """
                                docker build -t ${REGISTRY_USER}/${service}:${IMAGE_TAG} .
                                docker push ${REGISTRY_USER}/${service}:${IMAGE_TAG}
                            """
                        }
                    }
                }
            }
        }

        stage('Checkout GitOps Repo') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-token-for-gitops-repo',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh """
                        rm -rf ITC-backend-gitops
                        git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/BilalSaddiqueHundal/ITC-backend-gitops.git
                    """
                }
            }
        }

        stage('Update Image Tags in GitOps Repo') {
            steps {
                script {
                    def services = [
                        'connectionservice',
                        'apigateway'
                    ]

                    for (service in services) {
                        sh """
                            sed -i 's|image: docker.io/${REGISTRY_USER}/${service}:.*|image: docker.io/${REGISTRY_USER}/${service}:${IMAGE_TAG}|g' ITC-backend-gitops/dev/${service}/deployment.yaml
                        """
                    }
                }
            }
        }

        stage('Commit and Push GitOps Changes') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-token-for-gitops-repo',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    dir('ITC-backend-gitops') {
                        sh """
                            git config user.email "jenkins@example.com"
                            git config user.name "jenkins"

                            git add dev/
                            git commit -m "Update image tags to build ${IMAGE_TAG}" || echo "No changes to commit"

                            git push https://${GIT_USER}:${GIT_TOKEN}@github.com/BilalSaddiqueHundal/ITC-backend-gitops.git HEAD:main
                        """
                    }
                }
            }
        }
    }
}