pipeline {
    agent any

    options {
        timeout(time: 10, unit: 'MINUTES') // Tiempo máximo para la ejecución del pipeline
    }

    environment {
        NEXUS_URL = "http://nexus:8083"
        CREDENTIALS_ID = "nexus-credentials" // ID de las credenciales almacenadas en Jenkins
        IMAGE_NAME = "sumador" // Nombre de la imagen Docker
        IMAGE_TAG = "${env.BUILD_NUMBER}" // Etiqueta de la imagen basada en el número de build
        NEXUS_HOST = "nexus:8083" // Host y puerto de Nexus
        NEXUS_REPO = "docker-hosted" // Ruta del repositorio en Nexus
        //ARTIFACT_ID = "elbuo8/webapp:${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."
                sh """
                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                """
            }
        }

        // Seguridad 1: Análisis de dependencias Node.js
        stage('Security - npm audit') {
            steps {
                echo "Analizando vulnerabilidades en dependencias..."
                sh '''
                    docker run --rm -v \${PWD}:/app node:20-alpine sh -c "
                        cd /app && 
                        npm audit --audit-level=moderate || 
                        (echo 'Vulnerabilidades encontradas en dependencias' && exit 1)
                    "
                '''
            }
        }

        stage('Run tests') {
            steps {
                echo "Ejecutando tests unitarias..."
                sh "docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} npm test"
            }
        }
        
        // Seguridad 2: Escaneo de imagen con Trivy
        stage('Security - Trivy Scan') {
            steps {
                echo "Escaneando imagen Docker con Trivy..."
                sh """
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image \
                        --exit-code 1 \
                        --severity CRITICAL \
                        --no-progress \
                        ${IMAGE_NAME}:${IMAGE_TAG} || \
                """
            }
        }

        stage('Tag Docker Image') {
            steps {
                echo "Tagging Docker image for Nexus repository..."
                sh """
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} \
                        ${NEXUS_HOST}/${NEXUS_REPO}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} \
                        ${NEXUS_HOST}/${NEXUS_REPO}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy Image') {
            steps {
                script {
                    sh """
                        echo "${NEXUS_PASSWORD}" | docker login -u "${NEXUS_USERNAME}" \
                        --password-stdin ${NEXUS_URL}
                    """
                    sh """
                        docker push ${NEXUS_HOST}/${NEXUS_REPO}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${NEXUS_HOST}/${NEXUS_REPO}/${IMAGE_NAME}:latest
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up local Docker images..."
            sh """
                docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true
                docker rmi ${NEXUS_HOST}/${NEXUS_REPO}/${IMAGE_NAME}:${IMAGE_TAG} || true
                docker rmi ${NEXUS_HOST}/${NEXUS_REPO}/${IMAGE_NAME}:latest || true
                docker logout ${NEXUS_URL}
            """
        }
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Check the logs for details."
        }
    }
}