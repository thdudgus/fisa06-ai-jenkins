pipeline {
    agent any

    environment {
        DOCKER_CREDS = credentials('docker-hub') 
        IMAGE_NAME = 'fisa-app'
        DEPLOY_IP = '54.249.201.237'
        SSH_CREDS_ID = 'ec2-fastapi-server'
        CONTAINER_NAME = 'docker-fastapi'
        FULL_IMAGE_PATH = "${DOCKER_CREDS_USR}/${IMAGE_NAME}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/YeonjiKim0316/fisa06-ai-jenkins.git'
            }
        }

        stage('Build') {
            steps {
                // 빌드 전 로컬 찌꺼기(태그 없는 이미지) 정리
                sh "docker image prune -f"
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Tag & Push') {
            steps {
                sh "docker tag ${IMAGE_NAME} ${FULL_IMAGE_PATH}:${BUILD_NUMBER}"
                sh 'echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin'
                sh "docker push ${FULL_IMAGE_PATH}:${BUILD_NUMBER}"
            }
        }

        stage('Deploy') {
            steps {
                sshagent(credentials: ["${SSH_CREDS_ID}"]) {
                    script {
                        def sshOptions = "-o StrictHostKeyChecking=no ubuntu@${DEPLOY_IP}"
                        
                        // 1. 기존 컨테이너 제거
                        sh "ssh ${sshOptions} 'sudo docker rm -f ${CONTAINER_NAME} || true'"
                        
                        // 2. 새 컨테이너 실행
                        def runCmd = "sudo docker run --name ${CONTAINER_NAME} -e TZ=Asia/Seoul -p 80:8000 -d -t ${FULL_IMAGE_PATH}:${BUILD_NUMBER}"
                        sh "ssh ${sshOptions} '${runCmd}'"
                        
                        // 3. [추가] 원격 서버(FastAPI) 내 구버전 이미지 정리 (Surgical Cleanup)
                        // 현재 실행 중인 태그(${BUILD_NUMBER})를 제외한 동일 이미지명의 모든 구버전 삭제
                        echo "Cleaning up old images on remote server..."
                        sh """
                            ssh ${sshOptions} "sudo docker images ${FULL_IMAGE_PATH} --format '{{.Tag}} {{.ID}}' | 
                            grep -v '${BUILD_NUMBER}' | 
                            awk '{print \$2}' | 
                            xargs -r sudo docker rmi || true"
                        """
                        
                        // 4. [추가] 원격 서버 내 태그 없는(dangling) 이미지 완전 정리
                        sh "ssh ${sshOptions} 'sudo docker image prune -f'"
                    }
                }
            }
        }
    }

// 빌드 성공/실패 여부와 무관하게 항상 실행되는 정리 작업
    post {
        always {
            // 1. 현재 빌드에서 생성한 임시 태그(fisa-app:latest) 삭제
            sh "sudo docker rmi ${IMAGE_NAME} || true"
            
            // 2. 젠킨스 서버 내 구버전 이미지 정리 (Surgical Cleanup)
            // 로컬에 저장된 fisa-app 이미지 중 최신 3개만 남기고 전부 삭제
            echo "Cleaning up old images on Jenkins server..."
            sh """
                sudo docker images ${FULL_IMAGE_PATH} --format '{{.Tag}} {{.ID}}' | 
                sort -V -r | awk 'NR>3 {print \$2}' | 
                xargs -r sudo docker rmi || true
            """
            
            // 3. 로컬 빌드 중 생성된 태그 없는(dangling) 중간 레이어 이미지들 정리
            sh "sudo docker image prune -f || true"
            
        }
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed. Please check the logs."
        }
    }
}