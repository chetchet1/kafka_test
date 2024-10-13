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
                        // Kafka 설치 및 LoadBalancer IP 가져오기
                        bat """
                        helm repo add bitnami https://charts.bitnami.com/bitnami
                        helm repo update
                        helm install kafka bitnami/kafka -f %VALUES_FILE_PATH%
                        timeout /t 120 >nul
                        """

                        // PowerShell에서 LoadBalancer IP 가져오기
                        def loadBalancerIp = powershell(script: """
                            kubectl get svc kafka --output jsonpath='{.status.loadBalancer.ingress[0].ip}'
                        """, returnStdout: true).trim()
                        
                        // LoadBalancer IP 출력
                        echo "LoadBalancer IP: ${loadBalancerIp}"

                        // values.yaml 파일에서 LoadBalancer IP 대체
                        powershell """
                        (Get-Content '%VALUES_FILE_PATH%' -Raw) -replace '<LoadBalancer-IP>', '${loadBalancerIp}' | Set-Content '%VALUES_FILE_PATH%'
                        """

                        // 수정된 values.yaml 파일 출력
                        powershell "Get-Content '%VALUES_FILE_PATH%'"
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
