pipeline {
    agent any

    environment {
        IMAGE_NAME = 'securedev-tickets'
        PROD_HOST = '10.0.1.130'
        PROD_PATH = '/opt/securedev'
        ZAP_REPORTS = '/home/ubuntu/zap/reports'
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
                echo 'Construyendo imagen Docker'
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

                    for i in $(seq 1 20); do
                      if curl -fsS http://127.0.0.1:18000/health; then
                        echo "Prueba de salud aprobada"
                        exit 0
                      fi
                      sleep 3
                    done

                    echo "La aplicacion no respondio correctamente"
                    docker logs securedev-ci
                    exit 1
                '''
            }
            post {
                always {
                    sh 'docker rm -f securedev-ci 2>/dev/null || true'
                }
            }
        }

        stage('4 - OWASP ZAP') {
            steps {
                sh '''
                    mkdir -p ${ZAP_REPORTS}

                    docker run --rm \
                      --network host \
                      -v ${ZAP_REPORTS}:/zap/wrk/:rw \
                      ghcr.io/zaproxy/zaproxy:stable \
                      zap-baseline.py \
                      -t http://127.0.0.1:18000 \
                      -r reporte-zap-${BUILD_NUMBER}.html \
                      -I
                '''
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
                          --exclude='__pycache__' \
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
        success {
            echo 'CI/CD FINALIZADO CORRECTAMENTE'
        }
        failure {
            echo 'CI/CD FALLIDO. Revisar la etapa con error.'
        }
        always {
            echo 'Ejecucion del pipeline finalizada'
        }
    }
}
