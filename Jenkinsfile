pipeline {
    agent none

    environment {
        IMAGE_NAME      = "ghcr.io/vfalconer/vinlabs-python-app"
        IMAGE_TAG       = "latest"
        IMAGE           = "${IMAGE_NAME}:${IMAGE_TAG}"

        AZURE_RG        = "vinlabs"
        AZURE_APP       = "vinlabs-python-app"

        HEALTH_ENDPOINT = "/health"
        APP_PORT        = "8000"
    }

    stages {
        stage('Checkout') {
            agent any
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            agent { label 'docker' }
            steps {
                sh """
                echo 'Building image ${IMAGE}...'
                docker build -t ${IMAGE} .
                """
            }
        }

        stage('Push to GitHub Registry') {
            agent { label 'docker' }
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh """
                    echo 'Logging into GHCR...'
                    echo \$GITHUB_TOKEN | docker login ghcr.io -u vfalconer --password-stdin
                    echo 'Pushing ${IMAGE}...'
                    docker push ${IMAGE}
                    docker logout ghcr.io
                    """
                }
            }
        }

        stage('Configure Azure Web App') {
            agent any
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN'),
                                 azureServicePrincipal(credentialsId: 'azure-sp')]) {
                    sh """
                    echo 'Configuring container image on Azure Web App...'
                    az webapp config container set \
                      --resource-group ${AZURE_RG} \
                      --name ${AZURE_APP} \
                      --docker-custom-image-name ${IMAGE} \
                      --docker-registry-server-url https://ghcr.io \
                      --docker-registry-server-user vfalconer \
                      --docker-registry-server-password \$GITHUB_TOKEN

                    echo 'Setting WEBSITES_PORT=${APP_PORT}...'
                    az webapp config appsettings set \
                      --resource-group ${AZURE_RG} \
                      --name ${AZURE_APP} \
                      --settings WEBSITES_PORT=${APP_PORT}
                    """
                }
            }
        }

        stage('Health Check') {
            agent any
            steps {
                sh """
                echo 'Running health check...'
                curl -fsS https://${AZURE_APP}.azurewebsites.net${HEALTH_ENDPOINT} || (echo 'Health check failed' && exit 1)
                """
            }
        }
    }

    post {
        failure {
            echo "Deployment failed — tail logs: az webapp log tail --name ${AZURE_APP} --resource-group ${AZURE_RG}"
        }
        success {
            echo "Deployment succeeded — CBAOV Flask app is live on Azure!"
        }
    }
}
