pipeline {
    agent any

    environment {
        APP_NAME   = 'securedev-app'
        IMAGE_NAME = 'securedev-web:latest'
        TEST_PORT  = '5001'
        PROD_PORT  = '5000'
    }

    stages {
        stage('1. Clonar Código y Dependencias') {
            steps {
                echo '=== Descargando el código más reciente desde GitHub (SCM) ==='
                checkout scm
                sh '''
                    echo "Verificando archivos del repositorio y gestión de dependencias (Dependabot)..."
                    ls -la
                    if [ -f requirements.txt ]; then
                        echo "--- Dependencias declaradas en requirements.txt ---"
                        cat requirements.txt
                    fi
                '''
            }
        }

        stage('2. Construcción (Build Docker)') {
            steps {
                echo '=== Construyendo la imagen Docker de la aplicación ==='
                sh '''
                    docker build -t ${IMAGE_NAME} .
                    docker images | grep securedev-web
                '''
            }
        }

        stage('3. Pruebas de Seguridad (OWASP ZAP)') {
            steps {
                echo '=== Levantando entorno de pruebas y ejecutando OWASP ZAP ==='
                sh '''
                    # Eliminar contenedor de pruebas anterior si existe
                    docker rm -f ${APP_NAME}-test || true

                    # Desplegar contenedor temporal para análisis dinámico (DAST)
                    docker run -d --name ${APP_NAME}-test -p ${TEST_PORT}:5000 ${IMAGE_NAME}
                    sleep 5

                    # Obtener la IP interna del contenedor para que OWASP ZAP pueda escanearlo
                    TEST_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ${APP_NAME}-test)
                    echo "Ejecutando escaneo OWASP ZAP sobre http://${TEST_IP}:5000 ..."

                    # Crear carpeta de reportes con permisos de escritura para ZAP
                    mkdir -p zap-reports
                    chmod 777 zap-reports

                    # Ejecutar OWASP ZAP Baseline (-I permite generar el reporte aunque detecte alertas)
                    docker run --rm -v "${WORKSPACE}/zap-reports:/zap/wrk/:rw" \
                        ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
                        -t http://${TEST_IP}:5000 \
                        -r reporte_owasp_zap.html \
                        -I || true

                    # Limpiar contenedor temporal de pruebas
                    docker rm -f ${APP_NAME}-test || true
                '''
            }
        }

        stage('4. Despliegue a Producción') {
            steps {
                echo '=== Desplegando contenedor en Producción ==='
                sh '''
                    # Detener y reemplazar despliegue anterior en producción
                    docker rm -f ${APP_NAME}-prod || true

                    # Levantar contenedor en producción
                    docker run -d --name ${APP_NAME}-prod --restart unless-stopped -p ${PROD_PORT}:5000 ${IMAGE_NAME}
                    sleep 3

                    echo "Verificando estado del contenedor en producción:"
                    docker ps | grep ${APP_NAME}-prod
                '''
            }
        }
    }

    post {
        always {
            echo '=== Archivando reporte de OWASP ZAP para trazabilidad y auditoría ==='
            archiveArtifacts artifacts: 'zap-reports/reporte_owasp_zap.html', allowEmptyArchive: true
        }
        success {
            echo '¡Proceso finalizado! Análisis de seguridad completado y aplicación desplegada en Docker.'
        }
        failure {
            echo 'El Pipeline falló. Revisa el Console Output para identificar el error.'
        }
    }
}
