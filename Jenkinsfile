pipeline {
    agent any

    environment {
        IMAGE_NAME     = "ghcr.io/vfalconer/vinlabs-python-app"
        IMAGE_TAG      = "build-${BUILD_NUMBER}"
        IMAGE          = "${IMAGE_NAME}:${IMAGE_TAG}"
        IMAGE_LATEST   = "${IMAGE_NAME}:latest"

        AZURE_RG       = "vinlabs"
        AZURE_APP      = "vinlabs-python-app"
        HEALTH_ENDPOINT= "/"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t $IMAGE .
                docker tag $IMAGE $IMAGE_LATEST
                """
            }
        }

        stage('Push to GitHub Registry') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh """
                    echo \$GITHUB_TOKEN | docker login ghcr.io -u vfalconer --password-stdin
                    docker push $IMAGE
                    docker push $IMAGE_LATEST
                    docker logout ghcr.io
                    """
                }
            }
        }

        stage('Configure Azure Web App') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN'),
                                 azureServicePrincipal(credentialsId: 'azure-sp')]) {
                    sh """
                    # Set container image
                    az webapp config container set \
                      --resource-group $AZURE_RG \
                      --name $AZURE_APP \
                      --docker-custom-image-name $IMAGE_LATEST \
                      --docker-registry-server-url https://ghcr.io \
                      --docker-registry-server-user vfalconer \
                      --docker-registry-server-password \$GITHUB_TOKEN

                    # Set port for Flask/Gunicorn
                    az webapp config appsettings set \
                      --resource-group $AZURE_RG \
                      --name $AZURE_APP \
                      --settings WEBSITES_PORT=8000
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                sh """
                echo "Running health check..."
                curl -f https://$AZURE_APP.azurewebsites.net$HEALTH_ENDPOINT || exit 1
                """
            }
        }
    }

    post {
        failure {
            echo "❌ Deployment failed — tail logs: az webapp log tail --name $AZURE_APP --resource-group $AZURE_RG"
        }
        success {
            echo "✅ Deployment succeeded — CBAOV Flask app is live on Azure!"
        }
    }
}
