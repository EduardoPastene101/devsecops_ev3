
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'securedev-tickets'
        PROD_HOST = '10.0.1.130'
        PROD_PATH = '/opt/securedev'
        ZAP_REPORTS = '/home/ubuntu/zap/reports'
        TEST_CONTAINER = 'securedev-ci'
        TEST_PORT = '18000'
    }

    stages {

        stage('1 - Checkout') {
            steps {
                echo 'Obteniendo codigo desde GitHub'
                checkout scm
            }
        }

        stage('2 - Build') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                    docker image ls ${IMAGE_NAME}
                '''
            }
        }

        
		

		stage('3 - Tests') {
			steps {
				sh '''
					docker rm -f securedev-ci 2>/dev/null || true

					echo "Iniciando contenedor de pruebas..."

					docker run -d \
					  --name securedev-ci \
					  -p 18000:8000 \
					  ${IMAGE_NAME}:${BUILD_NUMBER}

					READY=0

					echo "Esperando que Flask inicie..."

					for i in $(seq 1 20); do
						if docker exec securedev-ci python -c \
						  "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=3).read().decode())"
						then
							READY=1
							break
						fi
						sleep 3
					done

					if [ "$READY" -ne 1 ]; then
						echo "ERROR: La aplicacion no respondio"
						docker logs securedev-ci
						exit 1
					fi

					echo "Prueba de salud aprobada"

					echo "Verificando metricas Prometheus..."

					docker exec securedev-ci python -c \
					  "import urllib.request; r=urllib.request.urlopen('http://127.0.0.1:8000/metrics'); data=r.read().decode(); assert 'securedev_http_requests_total' in data; print('Metricas disponibles correctamente')"

					echo "Verificando pagina de login..."

					docker exec securedev-ci python -c \
					  "import urllib.request; r=urllib.request.urlopen('http://127.0.0.1:8000/login'); assert r.status == 200; print('Login responde correctamente')"

					echo "Todas las pruebas funcionales finalizaron correctamente"
				'''
			}
		}



        stage('4 - OWASP ZAP') {
            steps {
                sh '''
                    mkdir -p ${ZAP_REPORTS}

                    docker rm -f zap-scan 2>/dev/null || true

                    docker run \
                      --name zap-scan \
                      --network host \
                      -v ${ZAP_REPORTS}:/zap/wrk/:rw \
                      ghcr.io/zaproxy/zaproxy:stable \
                      zap-baseline.py \
                      -t http://127.0.0.1:${TEST_PORT} \
                      -r reporte-zap-${BUILD_NUMBER}.html \
                      -I

                    docker cp zap-scan:/zap/wrk/reporte-zap-${BUILD_NUMBER}.html \
                      ${WORKSPACE}/reporte-zap-${BUILD_NUMBER}.html

                    docker rm zap-scan
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reporte-zap-*.html',
                                     allowEmptyArchive: true
                }
            }
        }

        stage('5 - Deploy') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'production-ssh',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        mkdir -p ~/.ssh
                        ssh-keyscan -H ${PROD_HOST} >> ~/.ssh/known_hosts

                        rsync -avz --delete \
                          --exclude='.git' \
                          --exclude='.env' \
                          --exclude='*.db' \
                          --exclude='*.sqlite' \
                          --exclude='*.sqlite3' \
                          --exclude='__pycache__' \
                          --exclude='.venv' \
                          --exclude='reporte-zap-*.html' \
                          -e "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=yes" \
                          ./ ${SSH_USER}@${PROD_HOST}:${PROD_PATH}/

                        ssh -i ${SSH_KEY} \
                          -o StrictHostKeyChecking=yes \
                          ${SSH_USER}@${PROD_HOST} \
                          "cd ${PROD_PATH} && docker compose -f compose.yaml up -d --build"
                    '''
                }
            }
        }

        stage('6 - Verificacion produccion') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'production-ssh',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        ssh -i ${SSH_KEY} \
                          -o StrictHostKeyChecking=yes \
                          ${SSH_USER}@${PROD_HOST} \
                          "curl -fsS http://localhost:8000/health"
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                docker rm -f ${TEST_CONTAINER} 2>/dev/null || true
                docker rm -f zap-scan 2>/dev/null || true
            '''
            echo 'Ejecucion del pipeline finalizada'
        }

        success {
            echo 'CI/CD FINALIZADO CORRECTAMENTE'
        }

        failure {
            echo 'CI/CD FALLIDO. Revisar la etapa con error.'
        }
    }
}
