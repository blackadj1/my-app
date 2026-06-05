pipeline {
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
    # 1. dind와 공유하는 로컬 소켓 볼륨을 마운트합니다.
    volumeMounts:
    - mountPath: /var/run
      name: docker-sock
  - name: dind
    image: docker:24-dind
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
    # 2. 로컬 소켓을 생성할 공유 볼륨을 마운트합니다.
    volumeMounts:
    - mountPath: /var/run
      name: docker-sock
  # 3. 컨테이너들이 내부 소켓 파일(docker.sock)을 공유할 수 있도록 emptyDir 선언
  volumes:
  - name: docker-sock
    emptyDir: {}
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
        
        // [수정 완료] 젠킨스 일감 이름(my-app)을 변수 정의 단계에서 100% 안전하게 직접 주입합니다.
        IMAGE_NAME        = "${env.JOB_BASE_NAME}"
    }
    
    stages {
        // 1단계: 빌드 및 ACR 푸시
        stage('Docker Build & Push to ACR') {
            steps {
                container('docker') {
                    sh """
                        # 1. ACR 관리자 계정으로 로그인 (로컬 소켓을 공유하므로 TCP 없이 완벽 작동)
                        docker login ${REGISTRY_URL} -u ${ACR_ADMIN_USER} -p ${ACR_ADMIN_PASS}
                        
                        # 2. 이미지 빌드 및 ACR 푸시
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
