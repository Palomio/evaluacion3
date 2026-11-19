pipeline {
    agent any

    environment {
        APP_NAME   = 'securedev-app'
        IMAGE_NAME = 'securedev-web:latest'
        TEST_PORT  = '5001'
        PROD_PORT  = '5050'
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
                    # Limpiar contenedores previos de pruebas si existen
                    docker rm -f ${APP_NAME}-test zap-scanner || true

                    # Desplegar contenedor temporal para análisis dinámico (DAST)
                    docker run -d --name ${APP_NAME}-test -p ${TEST_PORT}:5000 ${IMAGE_NAME}
                    sleep 5

                    # Obtener la IP interna del contenedor para escanearlo
                    TEST_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ${APP_NAME}-test)
                    echo "Ejecutando escaneo OWASP ZAP sobre http://${TEST_IP}:5000 ..."

                    # Ejecutar OWASP ZAP Baseline
                    docker run --name zap-scanner -u root \
                        ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
                        -t http://${TEST_IP}:5000 \
                        -r reporte_owasp_zap.html \
                        -I || true

                    # Extraer el reporte HTML directamente al workspace de Jenkins
                    mkdir -p zap-reports
                    docker cp zap-scanner:/zap/wrk/reporte_owasp_zap.html zap-reports/reporte_owasp_zap.html || true

                    # Limpiar contenedores temporales de pruebas
                    docker rm -f ${APP_NAME}-test zap-scanner || true
                '''
            }
        }

        stage('4. Despliegue a Producción') {
            steps {
                echo '=== Desplegando contenedor en Producción ==='
                sh '''
                    # Detener y reemplazar despliegue anterior en producción
                    docker rm -f ${APP_NAME}-prod || true

                    # Levantar contenedor en producción en el puerto 5050
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
