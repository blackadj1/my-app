pipeline {
    // 1. [핵심] 클라이언트(docker)와 백그라운드 데몬(dind) 컨테이너 분리 정의
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
        
        IMAGE_NAME        = ""
    }
    
    stages {
        // 1단계: 100% 성공하는 안전한 셸 명령어로 이미지 이름 추출
        stage('Initialize') {
            steps {
                script {
                    // git config와 sed를 결합해 깃 저장소 이름을 안정적으로 가져옵니다.
                    env.IMAGE_NAME = sh(script: "git config --get remote.origin.url | sed 's/.*\\///;s/\\.git//'", returnStdout: true).trim()
                    
                    echo "============================================="
                    echo "동적으로 감지된 이미지 이름: ${env.IMAGE_NAME}"
                    echo "============================================="
                }
            }
        }

        // 2단계: 분리된 'docker' 클라이언트 컨테이너를 사용하여 안전하게 빌드 및 ACR 푸시
        stage('Docker Build & Push to ACR') {
            steps {
                container('docker') { // ★ dind가 아닌 docker 클라이언트 컨테이너를 지정합니다.
                    sh """
                        apk add --no-cache git
                        docker login ${REGISTRY_URL} -u ${ACR_ADMIN_USER} -p ${ACR_ADMIN_PASS}
                        docker build -t ${env.IMAGE_NAME}:${IMAGE_TAG} .
                        docker tag ${env.IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY_URL}/${env.IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${REGISTRY_URL}/${env.IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        // 3단계: 배포용 Git 저장소 파일 자동 치환 후 푸시
        stage('Update ArgoCD Manifest') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-token', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh """
                        git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@${MANIFEST_REPO_URL} manifest-temp
                        cd manifest-temp
                        
                        # deployment.yaml의 이미지 정보 치환
                        sed -i "s|image: ${REGISTRY_URL}/${env.IMAGE_NAME}:.*|image: ${REGISTRY_URL}/${env.IMAGE_NAME}:${IMAGE_TAG}|g" ${env.IMAGE_NAME}/deployment.yaml
                        
                        git config user.email "jenkins-ci@yourdomain.com"
                        git config user.name "Jenkins CI Bot"
                        git add ${env.IMAGE_NAME}/deployment.yaml
                        git commit -m "Deploy: Update ${env.IMAGE_NAME} to tag ${IMAGE_TAG} [Jenkins Build #${BUILD_NUMBER}]"
                        git push origin main
                        
                        cd ..
                        rm -rf manifest-temp
                    """
                }
            }
        }
    }
}
