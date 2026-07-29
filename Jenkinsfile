pipeline {
    agent any

    tools {
        nodejs 'node24'
    }

    parameters {
        booleanParam(
            name: 'ENFORCE_ZAP_GATE',
            defaultValue: false,
            description: 'If true, High-risk ZAP alerts abort the pipeline. Juice Shop is deliberately full of High-risk findings (SQLi, XSS, etc.), so this is off by default to let the pipeline demonstrate its later stages; turn it on to see the strict gate behavior.'
        )
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

        stage('Install Docker CLI') {
            steps {
                sh '''
                    if command -v docker >/dev/null 2>&1; then
                        echo "Docker CLI already present"
                    else
                        echo "Docker CLI not found, installing..."
                        apt-get update -y
                        apt-get install -y ca-certificates curl gnupg
                        install -m 0755 -d /etc/apt/keyrings
                        curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
                        chmod a+r /etc/apt/keyrings/docker.asc
                        echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo $VERSION_CODENAME) stable" > /etc/apt/sources.list.d/docker.list
                        apt-get update -y
                        apt-get install -y docker-ce-cli
                    fi
                    docker --version
                '''
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
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('SCA - Snyk') {
            steps {
                sh 'mkdir -p ${REPORT_DIR}'
                sh '''
                    npx snyk auth ${SNYK_TOKEN}
                    npx snyk test --json-file-output=${REPORT_DIR}/snyk-report.json --severity-threshold=critical
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

        stage('Container Scan - Trivy') {
            steps {
                sh 'mkdir -p ${REPORT_DIR}'
                sh '''
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      -v ${WORKSPACE}/${REPORT_DIR}:/reports \
                      aquasec/trivy:latest image \
                      --format json \
                      --output /reports/trivy-report.json \
                      ${IMAGE_NAME}:${BUILD_NUMBER}

                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      aquasec/trivy:latest image \
                      --exit-code 1 \
                      --severity CRITICAL \
                      ${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/trivy-report.json", allowEmptyArchive: true
                }
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
                //
                // ENFORCE_ZAP_GATE=false (default) uses rules.tsv, where High-risk
                // rules are WARN, so zap-baseline.py never exits 2 (Juice Shop is
                // deliberately full of High-risk findings). ENFORCE_ZAP_GATE=true
                // uses rules-strict.tsv, where High-risk rules are FAIL, so the
                // stage aborts the pipeline on any High-risk alert.
                sh '''
                    mkdir -p ${WORKSPACE}/${REPORT_DIR}
                    if [ "${ENFORCE_ZAP_GATE}" = "true" ]; then
                        cp ${WORKSPACE}/.zap/rules-strict.tsv ${WORKSPACE}/${REPORT_DIR}/rules.tsv
                    else
                        cp ${WORKSPACE}/.zap/rules.tsv ${WORKSPACE}/${REPORT_DIR}/rules.tsv
                    fi

                    docker run --rm \
                      --network host \
                      -v ${WORKSPACE}/${REPORT_DIR}:/zap/wrk:rw \
                      ghcr.io/zaproxy/zaproxy:stable \
                      zap-baseline.py \
                        -t http://localhost:${APP_PORT} \
                        -r zap-report.html \
                        -J zap-report.json \
                        -c rules.tsv \
                        -a
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
