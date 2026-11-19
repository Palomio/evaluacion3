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
                echo '=== Levantando entorno de pruebas y ejecutando escaneo activo OWASP ZAP ==='
                sh '''
                    # Limpiar contenedores y volumen temporal previos
                    docker rm -f ${APP_NAME}-test zap-scanner || true
                    docker volume rm zap_wrk_vol || true
                    docker volume create zap_wrk_vol

                    # Desplegar contenedor temporal para análisis dinámico (DAST)
                    docker run -d --name ${APP_NAME}-test -p ${TEST_PORT}:5000 ${IMAGE_NAME}
                    sleep 5

                    # Obtener la IP interna del contenedor
                    TEST_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ${APP_NAME}-test)
                    echo "Ejecutando OWASP ZAP Full Scan sobre http://${TEST_IP}:5000 ..."

                    # Ejecutar escaneo activo con /zap/wrk montado y sesión activa
                    docker run --rm --name zap-scanner \
                        -v zap_wrk_vol:/zap/wrk \
                        ghcr.io/zaproxy/zaproxy:stable zap-full-scan.py \
                        -t http://${TEST_IP}:5000 \
                        -r reporte_owasp_zap.html \
                        -z "-config replacer.full_list(0).description=auth -config replacer.full_list(0).enabled=true -config replacer.full_list(0).matchtype=REQ_HEADER -config replacer.full_list(0).matchstr=Cookie -config replacer.full_list(0).regex=false -config replacer.full_list(0).replacement=user_session=admin" \
                        -I || true

                    # Extraer el reporte HTML desde el volumen al workspace de Jenkins
                    mkdir -p zap-reports
                    docker run --rm -v zap_wrk_vol:/zap/wrk ghcr.io/zaproxy/zaproxy:stable \
                        cat /zap/wrk/reporte_owasp_zap.html > zap-reports/reporte_owasp_zap.html || true

                    # Limpiar contenedor temporal y volumen
                    docker rm -f ${APP_NAME}-test || true
                    docker volume rm zap_wrk_vol || true
                '''
            }
        }

        stage('4. Despliegue a Producción') {
            steps {
                echo '=== Desplegando contenedor en Producción ==='
                sh '''
                    docker rm -f ${APP_NAME}-prod || true
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
