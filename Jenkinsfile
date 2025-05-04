pipeline {
    agent any

    parameters {
        string(name: 'GITHUB_BRANCH', defaultValue: 'main', description: '브랜치 이름')
    }

    environment {
        GITHUB_URL = "github.com/mooa-lee/test.git"
        BLOBSTORE = 'http://192.168.56.101:28080'
        DOCKER_REGISTRY = '192.168.56.101:5000'
        DOCKER_IMAGE = 'mendix-demo'
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        BUILD_PATH = 'result'
        DOCKER_BUILD_PACK_PATH = '/home/vagrant/docker-mendix-buildpack'
        DESTINATION = "${DOCKER_BUILD_PACK_PATH}/${BUILD_PATH}"
        APP_NODE1_HOSTNAME = 'deploy-node'
        APP_NODE2_HOSTNAME = 'app-node'
        APP_NODE1_IP = '192.168.56.101'
        APP_NODE2_IP = '192.168.56.102'
        APP_DB_NAME = 'mxdb'
        APP_DB_PORT = '5432'
        APP_PORT = '18080'
        APP_METRIC_PORT = '18082'
        METRICS_TYPE = 'micrometer'
    }

    stages {
        stage('Extract Repository Name') {
            steps {
                script {
                    REPO_NAME = GITHUB_URL.split('/')[-1].replace('.git', '')
                    echo "Repository Name Extracted: ${REPO_NAME}"
                }
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: params.GITHUB_BRANCH,
                    credentialsId: 'gat',
                    url: "https://${GITHUB_URL}"
                script {
                    checkoutCode()
                }
            }
        }

        stage('Prepare Build Context') {
            steps {
                script {
                    prepareBuildContext()
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    buildDockerImage()
                    tagDockerImage()
                    pushDockerImage()
                }
            }
        }

        stage('Deploy and Verify') {
            steps {
                script {
                    withAppCredentials {
                        def servers = getAppServers()
                        servers.each { server ->
                            deployToServer(server)
                            verifyServer(server)
                        }
                    }
                }
            }
        }

        stage('Cleanup Docker Images') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'app-server', usernameVariable: 'SSH_USER', passwordVariable: 'SSH_PASS')]) {
                        def servers = getAppServers()
                        servers.each { cleanupLocalDockerImages(it) }
                        cleanupRegistryDockerImages(servers[0])
                    }
                }
            }
        }
    }
}

///////////////////////
// === FUNCTIONS === //
///////////////////////

def getAppServers() {
    return [
        [name: env.APP_NODE2_HOSTNAME, host: env.APP_NODE2_IP, port: env.APP_PORT, containerSuffix: '']
    ]
}

def withAppCredentials(closure) {
    withCredentials([
        string(credentialsId: 'admin-pass', variable: 'ADMIN_PASS'),
        usernamePassword(credentialsId: 'app-server', usernameVariable: 'SSH_USER', passwordVariable: 'SSH_PASS'),
        usernamePassword(credentialsId: 'app-db', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')
    ]) {
        closure()
    }
}

def sshExecute(server, command) {
    sshCommand remote: [
        name: server.name,
        host: server.host,
        user: "${SSH_USER}",
        password: "${SSH_PASS}",
        allowAnyHosts: true
    ], command: command
}

def checkoutCode() {
    sh """
    echo "[✓] 기존 소스 디렉토리 삭제"
    sudo rm -rf ${DOCKER_BUILD_PACK_PATH}/${REPO_NAME}

    echo "[✓] 소스 디렉토리 복사"
    cp -r ${WORKSPACE} ${DOCKER_BUILD_PACK_PATH}/${REPO_NAME}

    sudo chown -R vagrant.vagrant ${DOCKER_BUILD_PACK_PATH}/${REPO_NAME}
    """
}

def prepareBuildContext() {
    sh """
    echo "[✓] 빌드 컨텍스트 초기화"
    sudo rm -rf ${DESTINATION}
    mkdir -p ${DESTINATION}
    sudo chown -R vagrant.vagrant ${DESTINATION}

    cd ${DOCKER_BUILD_PACK_PATH}
    sudo ./build.py --artifacts-repository ${DOCKER_REGISTRY}/mendix-buildpack --source ./${REPO_NAME} --destination ./${BUILD_PATH} build-mda-dir
    """
}

def buildDockerImage() {
    sh """
    echo "[✓] Docker 이미지 빌드 중..."
    sudo docker build \
        --build-arg BLOBSTORE='${BLOBSTORE}' \
        -t ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG} \
        -f ${DOCKER_BUILD_PACK_PATH}/Dockerfile \
        ${DESTINATION}
    """
}

def tagDockerImage() {
    sh """
    echo "[✓] latest 태그 추가"
    sudo docker tag ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:latest
    """
}

def pushDockerImage() {
    sh """
    echo "[✓] Docker 이미지 레지스트리에 푸시"
    sudo docker push ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG}
    sudo docker push ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:latest

    DANGLING=\$(sudo docker images -f "dangling=true" -q)
    if [ ! -z "\$DANGLING" ]; then
        echo "Dangling 이미지 제거"
        sudo docker rmi -f \$DANGLING
    fi
    """
}

def deployToServer(server) {
    sshExecute(server, """
        echo "[✓] 서버 ${server.name}에 이미지 배포"
        sudo docker pull ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG}

        if sudo docker ps -q --filter "name=${DOCKER_IMAGE}${server.containerSuffix}" | grep -q .; then
            echo "기존 컨테이너 중지 및 제거"
            sudo docker stop ${DOCKER_IMAGE}${server.containerSuffix}
            sudo docker rm ${DOCKER_IMAGE}${server.containerSuffix}
        fi

        echo "새 컨테이너 실행"
        sudo docker run -d --name ${DOCKER_IMAGE}${server.containerSuffix} \
            --restart unless-stopped \
            -e DATABASE_ENDPOINT=postgres://${DB_USER}:${DB_PASS}@${APP_NODE2_IP}:${APP_DB_PORT}/${APP_DB_NAME} \
            -e ADMIN_PASSWORD=${ADMIN_PASS} \
            -e MXRUNTIME_com_mendix_metrics_Type=${METRICS_TYPE} \
            -e MXRUNTIME_Metrics_Registries='[{"type": "prometheus", "settings": {"step": "1m"}}]' \
            -e MXRUNTIME_Metrics_ApplicationTags='{"hostId": "${server.name}", "environment": "${BUILD_PATH}"}' \
            -p ${server.port}:18080 \
            -p ${APP_METRIC_PORT}:18082 \
            ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG}
    """)
}

def verifyServer(server) {
    def retries = 10
    def success = false
    for (int i = 0; i < retries; i++) {
        echo "[✓] ${server.name} 상태 점검 시도 ${i + 1}/${retries}"
        def result = sh(script: "curl -sf http://${server.host}:${server.port}/", returnStatus: true)
        if (result == 0) {
            echo "[✓] ${server.name} 애플리케이션 확인 완료"
            success = true
            break
        }
        sleep(5)
    }
    if (!success) {
        error "[✗] ${server.name}에 애플리케이션이 정상적으로 배포되지 않았습니다."
    }
}

def cleanupLocalDockerImages(server) {
    sshExecute(server, """
        echo "[🧹] ${server.name} - 로컬 이미지 정리"
        old_images=\$(sudo docker images ${DOCKER_REGISTRY}/${DOCKER_IMAGE} --format '{{.Tag}}' | grep -v latest | sort -nr | tail -n +4)
        for tag in \$old_images; do
            echo "Removing local image: \$tag"
            sudo docker rmi -f ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:\$tag
        done
    """)
}

def cleanupRegistryDockerImages(server) {
    sshExecute(server, """
        echo "[🧹] ${server.name} - 레지스트리 이미지 정리"
        tags=\$(curl -s http://${DOCKER_REGISTRY}/v2/${DOCKER_IMAGE}/tags/list | jq -r '.tags[]' | grep -v 'latest' | sort -nr)
        count=0
        tags_to_delete=""
        for tag in \$tags; do
            count=\$((count+1))
            if [ \$count -gt 3 ]; then
                tags_to_delete="\$tags_to_delete \$tag"
            fi
        done
        for tag in \$tags_to_delete; do
            digest=\$(curl -sI -H "Accept: application/vnd.docker.distribution.manifest.v2+json" http://${DOCKER_REGISTRY}/v2/${DOCKER_IMAGE}/manifests/\$tag | grep Docker-Content-Digest | awk '{print \$2}' | tr -d '\\r')
            if [ -n "\$digest" ]; then
                curl -s -X DELETE http://${DOCKER_REGISTRY}/v2/${DOCKER_IMAGE}/manifests/\$digest
                echo "Deleted \$tag from registry"
            fi
        done
    """)
}
