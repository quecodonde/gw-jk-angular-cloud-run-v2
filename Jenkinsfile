pipeline {
    agent any

    environment {
        // Variables migradas de tu env de GitHub Actions
        GCP_PROJECT_ID         = "project-f5b5f7ba-7267-405d-99c"
        GCP_REGION             = "us-central1"
        ARTIFACT_REGISTRY_REPO = "gw-jk-artifact-registry-01" 
        CLOUD_RUN_SERVICE_NAME = "gemini-angular-app"
        
        // El secret ENVIRONMENT_NAME lo manejamos de forma segura en Jenkins
        // ENVIRONMENT_NAME       = credentials('ENVIRONMENT_NAME')
        ENVIRONMENT_NAME       = "dev"

        
        // Construimos el tag usando el ID de commit corto de Git
        GIT_COMMIT_SHORT       = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        IMAGE_TAG              = "${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT_ID}/${ARTIFACT_REGISTRY_REPO}/${CLOUD_RUN_SERVICE_NAME}:${GIT_COMMIT_SHORT}"
    }

    stages {
        stage('Checkout') {
            steps {
                // Jenkins realiza el checkout automáticamente si es un Pipeline de SCM,
                // pero lo aseguramos explícitamente aquí.
                checkout scm
            }
        }

        stage('GCP & Docker Auth') {
            steps {
                script {
                    // Configura gcloud para usar el proyecto correcto de manera global en este build
                    sh "gcloud config set project ${GCP_PROJECT_ID}"
                    
                    // Autentica Docker contra Artifact Registry usando la identidad nativa de la VM
                    sh "gcloud auth configure-docker ${GCP_REGION}-docker.pkg.dev --quiet"
                }
            }
        }

        stage('Build and Push Image') {
            steps {
                sh """
                docker build -t ${IMAGE_TAG} .
                docker push ${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to Cloud Run') {
            steps {
                // Despliegue usando las mismas banderas de tu GitHub Action
                sh """
                gcloud run deploy ${CLOUD_RUN_SERVICE_NAME}-${ENVIRONMENT_NAME} \
                    --image ${IMAGE_TAG} \
                    --region ${GCP_REGION} \
                    --platform managed \
                    --allow-unauthenticated \
                    --set-env-vars="ENVIRONMENT_NAME=${ENVIRONMENT_NAME}"
                """
            }
        }
    }

    post {
        always {
            // Limpieza opcional de la imagen local para no llenar el disco de la VM de Jenkins
            sh "docker rmi ${IMAGE_TAG} || true"
        }
    }
}
