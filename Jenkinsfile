pipeline {
    agent any

    options {
        timeout(time: 10, unit: 'MINUTES')
    }

    environment {
        NEXUS_URL      = "http://nexus:8083"
        CREDENTIALS_ID = "nexus-credentials"
        IMAGE_NAME     = "sumador"
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
        NEXUS_HOST     = "nexus:8083"
    }

    stages {

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Security - npm audit') {
            steps {
                echo "Analizando vulnerabilidades en dependencias..."
                sh """
                    docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} \
                        npm audit --audit-level=critical || \
                    (echo 'Vulnerabilidades criticas en dependencias' && exit 1)
                """
            }
        }

        stage('Run tests') {
            steps {
                echo "Ejecutando tests unitarios..."
                sh "docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} npm test"
            }
        }

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
                    (echo 'Vulnerabilidades CRITICAS en la imagen' && exit 1)
                """
            }
        }

        stage('Tag Docker Image') {
            steps {
                echo "Etiquetando imagen para Nexus..."
                sh """
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${NEXUS_HOST}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${NEXUS_HOST}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.CREDENTIALS_ID,
                    usernameVariable: 'NEXUS_USERNAME',
                    passwordVariable: 'NEXUS_PASSWORD'
                )]) {
                    sh """
                        echo \$NEXUS_PASSWORD | docker login -u \$NEXUS_USERNAME \
                            --password-stdin ${NEXUS_URL}
                        docker push ${NEXUS_HOST}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${NEXUS_HOST}/${IMAGE_NAME}:latest
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Limpiando imagenes locales..."
            sh """
                docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true
                docker rmi ${NEXUS_HOST}/${IMAGE_NAME}:${IMAGE_TAG} || true
                docker rmi ${NEXUS_HOST}/${IMAGE_NAME}:latest || true
                docker logout ${NEXUS_URL} || true
            """
        }
        success {
            echo "Pipeline completado exitosamente"
        }
        failure {
            echo "Pipeline fallo. Revisar los logs."
        }
    }
}