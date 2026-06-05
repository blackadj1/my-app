pipeline {
    // 1. [AKS 표준 설정] 컨테이너 분리 및 vfs 드라이버 적용
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: agent
spec:
  containers:
  - name: docker
    image: docker:24
    command:
    - cat
    tty: true
    env:
    - name: DOCKER_HOST
      value: tcp://localhost:2375
  - name: dind
    image: docker:24-dind
    securityContext:
      privileged: true
    command:
    - dockerd-entrypoint.sh
    args:
    - --storage-driver=vfs
    - --host=tcp://0.0.0.0:2375
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
'''
        }
    }
    
    environment {
        ACR_NAME          = 'acrenfinitymgprd'
        REGISTRY_URL      = "acrenfinitymgprd.azurecr.io"
        IMAGE_TAG         = "${BUILD_NUMBER}"
        
        ACR_ADMIN_USER    = 'acrenfinitymgprd' 
        ACR_ADMIN_PASS    = 'BgPnQOHC738hR6u2XX0POyrfdLkzrbIT0QJ1OeJ6E7kFnipEjH4KJQQJ99CEACNns7REqg7NAAACAZCRBGJH'

        MANIFEST_REPO_URL = 'github.com/blackadj1/manifest-repo-test.git'
        
        // [수정 완료] 젠킨스 일감 이름인 'my-app'을 시스템 전역 변수에서 바로 주입합니다. (에러 원천 차단)
        IMAGE_NAME        = "${env.JOB_BASE_NAME}"
    }
    
    stages {
        // 1단계: 빌드 및 ACR 푸시
        stage('Docker Build & Push to ACR') {
            steps {
                container('docker') {
                    sh """
                        # [대기 루프] 알핀 Linux에서도 100% 호환되는 POSIX while 문법 적용
                        echo "============================================="
                        echo "Docker 데몬이 준비될 때까지 대기합니다..."
                        echo "============================================="
                        attempt=1
                        while [ \$attempt -le 15 ]; do
                            if docker info >/dev/null 2>&1; then
                                echo "Docker 데몬이 성공적으로 준비되었습니다!"
                                break
                            fi
                            echo "Docker 데몬 응답 없음 (시도 \${attempt}/15). 2초 후 재시도..."
                            attempt=\$((attempt + 1))
                            sleep 2
                        done

                        if [ \$attempt -gt 15 ]; then
                            echo "에러: 30초 동안 Docker 데몬이 작동하지 않았습니다."
                            exit 1
                        fi

                        # ACR 로그인 및 이미지 빌드/푸시
                        docker login ${REGISTRY_URL} -u ${ACR_ADMIN_USER} -p ${ACR_ADMIN_PASS}
                        docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        // 2단계: 배포용 Git 저장소 파일 자동 치환 후 푸시
        stage('Update ArgoCD Manifest') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-token', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh """
                        git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@${MANIFEST_REPO_URL} manifest-temp
                        cd manifest-temp
                        
                        # deployment.yaml의 이미지 정보 치환
                        sed -i "s|image: ${REGISTRY_URL}/${IMAGE_NAME}:.*|image: ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}|g" ${IMAGE_NAME}/deployment.yaml
                        
                        git config user.email "jenkins-ci@yourdomain.com"
                        git config user.name "Jenkins CI Bot"
                        git add ${IMAGE_NAME}/deployment.yaml
                        git commit -m "Deploy: Update ${IMAGE_NAME} to tag ${IMAGE_TAG} [Jenkins Build #${BUILD_NUMBER}]"
                        git push origin main
                        
                        cd ..
                        rm -rf manifest-temp
                    """
                }
            }
        }
    }
}
