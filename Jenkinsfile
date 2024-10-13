pipeline {
    agent any

    environment {
        KUBECONFIG = "C:\\Windows\\system32\\config\\systemprofile\\.kube\\config"
        REGION = 'ap-northeast-2'
        AWS_CREDENTIAL_NAME = 'aws-key'
        VALUES_FILE_PATH = 'E:/docker_Logi/infra_structure/values.yaml'
    }

    stages {
        stage('Install Zookeeper and Kafka with Helm') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CREDENTIAL_NAME]]) {
                        // Helm 설치 및 대기
                        bat """
                            helm repo add bitnami https://charts.bitnami.com/bitnami
                            helm repo update
                            helm install kafka bitnami/kafka -f ${VALUES_FILE_PATH}
                            timeout /t 120 >nul
                        """

                        // LoadBalancer IP를 가져오기 위한 PowerShell 실행
                        def loadBalancerIp = bat(script: "powershell -Command \"kubectl get svc kafka --output jsonpath='{.status.loadBalancer.ingress[0].ip}'\"", returnStdout: true).trim()

                        // LoadBalancer IP를 파일에 저장
                        writeFile(file: 'LoadBalancerIP.txt', text: loadBalancerIp)
                        echo "LoadBalancer IP: ${loadBalancerIp}"

                        // values.yaml 파일의 LoadBalancer IP 업데이트
                        powershell """
                            (Get-Content '${VALUES_FILE_PATH}' -Raw) -replace '<LoadBalancer-IP>', '${loadBalancerIp}' | Set-Content '${VALUES_FILE_PATH}'
                        """

                        // values.yaml 파일 내용 확인
                        echo "Updated values.yaml: "
                        powershell "Get-Content '${VALUES_FILE_PATH}'"
                    }
                }
            }
        }

        stage('Upgrade Kafka') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CREDENTIAL_NAME]]) {
                        bat '''
                        helm upgrade kafka bitnami/kafka -f %VALUES_FILE_PATH%
                        kubectl rollout status deployment/kafka
                        kubectl get pods -l app.kubernetes.io/name=kafka
                        '''
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Kafka successfully installed and upgraded with external access!'
        }
        failure {
            echo 'Kafka installation or upgrade failed.'
        }
    }
}
