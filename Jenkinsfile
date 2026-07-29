pipeline {
    agent any

    tools {
        nodejs 'node24'
    }

    environment {
        SONAR_TOKEN     = credentials('sonar-token')
        SNYK_TOKEN      = credentials('snyk-token')
        IMAGE_NAME      = 'juice-shop-local'
        CONTAINER_NAME  = 'juice-shop-dast'
        APP_PORT        = '3000'
        REPORT_DIR      = 'reports'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install dependencies') {
            steps {
                sh 'npm install --include=dev'
            }
        }

        stage('SAST - SonarQube') {
            steps {
                withSonarQubeEnv('sonarcloud') {
                    sh '''
                        npx sonar-scanner \
                          -Dsonar.projectKey=PadmanabhanSaravanan_juiceshop \
                          -Dsonar.organization=padmanabhansaravanan \
                          -Dsonar.host.url=https://sonarcloud.io \
                          -Dsonar.token=${SONAR_TOKEN}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('SCA - Snyk') {
            steps {
                sh 'mkdir -p ${REPORT_DIR}'
                sh '''
                    npx snyk auth ${SNYK_TOKEN}
                    npx snyk test --json-file-output=${REPORT_DIR}/snyk-report.json || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/snyk-report.json", allowEmptyArchive: true
                }
            }
        }

        stage('Build Docker image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .'
            }
        }

        stage('Run app for DAST') {
            steps {
                sh '''
                    docker rm -f ${CONTAINER_NAME} || true
                    docker run -d --name ${CONTAINER_NAME} -p ${APP_PORT}:3000 ${IMAGE_NAME}:${BUILD_NUMBER}

                    echo "Waiting for Juice Shop to become ready..."
                    for i in $(seq 1 30); do
                        if curl -sf http://localhost:${APP_PORT} > /dev/null; then
                            echo "App is up"
                            break
                        fi
                        sleep 2
                    done
                '''
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                // ZAP writes/reads files relative to /zap/wrk, so the whole
                // report dir (not a subpath) must be the mount target, and
                // rules.tsv must be copied inside it first.
                sh '''
                    mkdir -p ${WORKSPACE}/${REPORT_DIR}
                    cp ${WORKSPACE}/.zap/rules.tsv ${WORKSPACE}/${REPORT_DIR}/rules.tsv

                    docker run --rm \
                      --network host \
                      -v ${WORKSPACE}/${REPORT_DIR}:/zap/wrk:rw \
                      ghcr.io/zaproxy/zaproxy:stable \
                      zap-baseline.py \
                        -t http://localhost:${APP_PORT} \
                        -r zap-report.html \
                        -J zap-report.json \
                        -c rules.tsv \
                        -a || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/zap-report.*", allowEmptyArchive: true
                    publishHTML(target: [
                        reportDir: "${REPORT_DIR}",
                        reportFiles: 'zap-report.html',
                        reportName: 'ZAP DAST Report',
                        keepAll: true,
                        alwaysLinkToLastBuild: true
                    ])
                }
            }
        }
    }

    post {
        always {
            sh 'docker rm -f ${CONTAINER_NAME} || true'
        }
    }
}
