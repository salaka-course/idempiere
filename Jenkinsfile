pipeline {

    agent any

    parameters {
        booleanParam(name: 'SKIP_MAVEN_BUILD', defaultValue: false, description: 'Sauter le build Maven')
        booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Activer le déploiement')
        string(name: 'DEPLOY_HOST', defaultValue: '192.168.2.183', description: 'IP du serveur de déploiement')
    }

    environment {
        PATH        = "/opt/maven/bin:${env.PATH}"
        REPO_URL    = 'https://github.com/salaka-course/idempiere.git'
        BRANCH      = 'release-12'
        IMAGE_NAME  = 'germain24/idempiere-release12'
        IMAGE_TAG   = "${BUILD_NUMBER}"
        IMAGE_FULL  = "germain24/idempiere-release12:${BUILD_NUMBER}"
        IMAGE_LATEST= 'germain24/idempiere-release12:latest'
        SONAR_PROJECT_KEY = 'idempiere-release12'
        DEPLOY_USER = 'isnov-promote'
        DEPLOY_DIR  = '/home/isnov-promote/idempiere-release12'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 120, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                echo "==> Checkout iDempiere ${BRANCH}"
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${BRANCH}"]],
                    extensions: [
                        [$class: 'CloneOption', shallow: true, depth: 1, timeout: 30]
                    ],
                    userRemoteConfigs: [[
                        url: 'https://github.com/salaka-course/idempiere.git',
                        credentialsId: 'GitHub Personal Access Token'
                    ]]
                ])
            }
        }

        stage('Trivy FS Scan') {
            steps {
                echo "==> Scan Trivy filesystem"
                sh """
                    trivy fs . \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --scanners vuln,secret \
                        --format table \
                        --no-progress \
                        --timeout 10m \
                        2>&1 | tee trivy-fs-report.txt || true
                """
            }
        }

        stage('Maven Build') {
            when {
                expression { !params.SKIP_MAVEN_BUILD }
            }
            steps {
                echo "==> Build Maven iDempiere"
                sh """
                    mvn clean verify \
                        -DskipTests=true \
                        --batch-mode \
                        --no-transfer-progress
                """
            }
            post {
                always {
                    junit allowEmptyResults: true,
                          testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Find Artifacts') {
            steps {
                echo "==> Recherche des artefacts Maven produits"
                sh """
                    echo "=== Contenu org.idempiere.p2/target/ ==="
                    ls -la org.idempiere.p2/target/ || echo "Dossier absent"

                    echo "=== Recherche répertoires produits ==="
                    find . -maxdepth 5 \
                        \\( -type d -name "*idempiere*server*" \
                        -o -type d -name "*gtk*linux*" \
                        -o -type d -name "org.adempiere.server*" \\) \
                        2>/dev/null | grep -v ".git" | grep -v "/.m2/"
                """
            }
        }

        stage('Docker Build') {
            steps {
                echo "==> Build image Docker ${IMAGE_FULL}"
                sh """
                    docker build \
                        --build-arg BUILD_NUMBER=${BUILD_NUMBER} \
                        --build-arg BRANCH=${BRANCH} \
                        -t ${IMAGE_FULL} \
                        -t ${IMAGE_LATEST} \
                        -f Dockerfile \
                        .
                """
            }
        }

        stage('Trivy Image Scan') {
            steps {
                echo "==> Scan Trivy image Docker"
                sh """
                    trivy image \
                        --exit-code 1 \
                        --severity CRITICAL \
                        --format table \
                        --no-progress \
                        --timeout 10m \
                        ${IMAGE_FULL} \
                        2>&1 | tee trivy-image-report.txt
                """
            }
        }

        stage('Push Docker Hub') {
            steps {
                echo "==> Push ${IMAGE_FULL} vers Docker Hub"
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push ${IMAGE_FULL}
                        docker push ${IMAGE_LATEST}
                        docker logout
                    """
                }
            }
        }

        stage('Deploy') {
            when {
                expression { params.DEPLOY == true }
            }
            steps {
                echo "==> Déploiement sur ${params.DEPLOY_HOST}"
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'deploy-ssh-key', keyFileVariable: 'SSH_KEY'),
                    string(credentialsId: 'db-password', variable: 'DB_PASS')
                ]) {
                    sh """
                        scp -i ${SSH_KEY} -o StrictHostKeyChecking=no \
                            docker-compose.yml \
                            ${DEPLOY_USER}@${params.DEPLOY_HOST}:${DEPLOY_DIR}/docker-compose.yml
                    """
                    sh """
                        ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no \
                            ${DEPLOY_USER}@${params.DEPLOY_HOST} bash -s << 'ENDSSH'
set -e
cd ${DEPLOY_DIR}
cat > .env << EOF
IMAGE_NAME=${IMAGE_FULL}
DB_HOST=localhost
DB_PORT=5444
DB_NAME=idempiere
DB_USER=adempiere
DB_PASSWORD=${DB_PASS}
APP_PORT=8099
EOF
grep -v PASSWORD .env
docker compose pull
docker compose up -d --remove-orphans
ENDSSH
                    """
                }
            }
        }

        stage('Health Check') {
            when {
                expression { params.DEPLOY == true }
            }
            steps {
                echo "==> Health Check iDempiere"
                sh """
                    sleep 30
                    curl --retry 5 --retry-delay 15 --retry-connrefused -f \
                        http://${params.DEPLOY_HOST}:8099/webui/index.zul \
                    && echo "==> iDempiere UP" \
                    || (echo "==> iDempiere ne répond pas" && exit 1)
                """
            }
        }
    }

    post {
        success {
            echo "==> Pipeline terminé avec succès — image : ${IMAGE_FULL}"
        }
        failure {
            echo "==> Pipeline en échec"
        }
        always {
            archiveArtifacts artifacts: 'trivy-*.txt', allowEmptyArchive: true
            sh """
                docker rmi ${IMAGE_FULL} ${IMAGE_LATEST} || true
                docker image prune -f || true
            """
//           cleanWs()
        }
    }
}