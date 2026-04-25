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
                    docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} \
                    npm audit --audit-level=critical || \
                    (echo "Vulnerabilidades críticas en dependencias" && exit 1)
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
                    (echo "Vulnerabilidades críticas en la imagen Docker" && exit 1)
                """
            }
        }

        stage('Tag Docker Image') {
            steps {
                echo "Tagging Docker image for Nexus repository..."
                sh """
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} \
                    ${NEXUS_HOST}/${IMAGE_NAME}:${IMAGE_TAG}
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} \
                    ${NEXUS_HOST}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy Image') {
            steps {
                script {
                    docker.withRegistry("${NEXUS_URL}", "${CREDENTIALS_ID}") {
                        def dockerImage = docker.image("${IMAGE_NAME}:${IMAGE_TAG}")
                        dockerImage.push("${IMAGE_TAG}")
                        dockerImage.push("latest")
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up local Docker images..."
            sh """
                docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true
                docker rmi ${NEXUS_HOST}/${IMAGE_NAME}:${IMAGE_TAG} || true
                docker rmi ${NEXUS_HOST}/${IMAGE_NAME}:latest || true
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