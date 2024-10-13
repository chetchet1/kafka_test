pipeline {
    agent any

    environment {
        KUBECONFIG = "C:\\Windows\\system32\\config\\systemprofile\\.kube\\config"
        REGION = 'ap-northeast-2'
        AWS_CREDENTIAL_NAME = 'aws-key'
        VALUES_FILE_PATH = 'E:/docker_Logi/infra_structure/values.yaml'
    }

    stages {
        // Install Zookeeper and Kafka with Helm
        stage('Install Zookeeper and Kafka with Helm') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CREDENTIAL_NAME]]) {
                        bat '''
                        helm repo add bitnami https://charts.bitnami.com/bitnami
                        helm repo update
                        
                        REM Install Kafka with initial settings
                        helm install kafka bitnami/kafka -f %VALUES_FILE_PATH%

                        REM Wait for Kafka service to be available
                        timeout /t 120 >nul

                        REM Get LoadBalancer IP using PowerShell
                        powershell -Command "kubectl get svc kafka --output jsonpath='{.status.loadBalancer.ingress[0].ip}'" > LoadBalancerIP.txt
                        type LoadBalancerIP.txt  REM 디버깅 출력: IP 주소를 출력합니다.
                        set /p LOAD_BALANCER_IP=<LoadBalancerIP.txt

                        REM Update advertised listeners in values.yaml
                        powershell -Command "(Get-Content '%VALUES_FILE_PATH%' -Raw) -replace '<LoadBalancer-IP>', '${LOAD_BALANCER_IP}' | Set-Content '%VALUES_FILE_PATH%'"
                        '''
                    }
                }
            }
        }

        // Upgrade Kafka with new configuration
        stage('Upgrade Kafka') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CREDENTIAL_NAME]]) {
                        bat '''
                        REM Upgrade Kafka with the updated values.yaml
                        helm upgrade kafka bitnami/kafka -f %VALUES_FILE_PATH%

                        REM Wait for the Kafka deployment to rollout
                        kubectl rollout status deployment/kafka

                        REM Check Kafka pods status
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
