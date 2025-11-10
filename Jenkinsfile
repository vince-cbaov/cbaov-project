pipeline {
    agent any
    environment {
        IMAGE = "ghcr.io/vfalconer/vinlabs-python-app:latest"
        AZURE_RG = "vinlabs"
        AZURE_APP = "vinlabs-python-app"
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE .'
            }
        }
        stage('Push to GitHub Registry') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh 'echo $GITHUB_TOKEN | docker login ghcr.io -u vfalconer --password-stdin'
                    sh 'docker push $IMAGE'
                }
            }
        }
        stage('Deploy to Azure Web App') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: 'azure-sp')]) {
                    sh '''
                    az webapp config container set \
                      --resource-group $AZURE_RG \
                      --name $AZURE_APP \
                      --docker-custom-image-name $IMAGE \
                      --docker-registry-server-url https://ghcr.io
                    '''
                }
            }
        }
        stage('Health Check') {
            steps {
                script {
                    sh '''
                    echo "Running health check..."
                    curl -f https://$AZURE_APP.azurewebsites.net/health || exit 1
                    '''
                }
            }
        }
    }
    post {
        failure {
            echo "Deployment failed — check logs with: az webapp log tail --name $AZURE_APP --resource-group $AZURE_RG"
        }
        success {
            echo "Deployment succeeded — CBAOV Flask app is live on Azure!"
        }
    }
}
