
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

					docker run -d \
					  --name securedev-ci \
					  -p 18000:8000 \
					  ${IMAGE_NAME}:${BUILD_NUMBER}

					READY=0

					for i in $(seq 1 20); do
						if curl -fsS http://127.0.0.1:18000/health; then
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

					echo "Verificando endpoint /metrics"
					curl -fsS http://127.0.0.1:18000/metrics \
					  | grep -q "securedev_http_requests_total"

					echo "Verificando pagina de login"
					curl -fsS http://127.0.0.1:18000/login \
					  -o /dev/null

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
