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
                        bat """
                        helm repo add bitnami https://charts.bitnami.com/bitnami
                        helm repo update
                        
                        helm install kafka bitnami/kafka -f %VALUES_FILE_PATH%

                        timeout /t 120 >nul

                        powershell -Command "kubectl get svc kafka --output jsonpath='{.status.loadBalancer.ingress[0].ip}'" > LoadBalancerIP.txt
                        type LoadBalancerIP.txt
                        set /p LOAD_BALANCER_IP=<LoadBalancerIP.txt

                        powershell -Command "(Get-Content '%VALUES_FILE_PATH%' -Raw) -replace '<LoadBalancer-IP>', '${LOAD_BALANCER_IP}' | Set-Content '%VALUES_FILE_PATH%'"

                        type %VALUES_FILE_PATH%
                        """
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