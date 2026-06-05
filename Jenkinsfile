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
        stage('Initialize') {
            steps {
                script {
                    // [수정] 리눅스 버전에 구애받지 않고, 젠킨스가 내부적으로 안전하게 깃 주소에서 레포명을 추출합니다.
                    def gitUrl = scm.getUserRemoteConfigs()[0].getUrl()
                    env.IMAGE_NAME = gitUrl.substring(gitUrl.lastIndexOf('/') + 1).replace('.git', '')
                    
                    echo "============================================="
                    echo "동적으로 감지된 이미지 이름: ${env.IMAGE_NAME}"
                    echo "============================================="
                }
            }
        }

        stage('Docker Build & Push to ACR') {
            steps {
                container('dind') {
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

        stage('Update ArgoCD Manifest') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-token', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh """
                        git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@${MANIFEST_REPO_URL} manifest-temp
                        cd manifest-temp
                        
                        # [수정] 정확히 치환될 수 있도록 큰따옴표 형식을 보완했습니다.
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
