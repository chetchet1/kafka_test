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
                        // Kafka 설치
                        bat """
                        helm repo add bitnami https://charts.bitnami.com/bitnami
                        helm repo update
                        helm install kafka bitnami/kafka -f ${VALUES_FILE_PATH}
                        """

                        // PowerShell에서 LoadBalancer IP 가져오기
                        def loadBalancerIp = powershell(script: """
                            Start-Sleep -Seconds 60  # 60초 대기
                            kubectl get svc kafka --output jsonpath='{.status.loadBalancer.ingress[0].ip}'
                        """, returnStdout: true).trim()
                        
                        if (loadBalancerIp) {
                            echo "LoadBalancer IP: ${loadBalancerIp}"
                        } else {
                            error "LoadBalancer IP could not be retrieved."
                        }

                        // values.yaml 파일에서 LoadBalancer IP 대체
                        powershell """
                        (Get-Content '$env:VALUES_FILE_PATH' -Raw) -replace '<LoadBalancer-IP>', '${loadBalancerIp}' | Set-Content '$env:VALUES_FILE_PATH'
                        """

                        // 수정된 values.yaml 파일 출력
                        powershell "Get-Content '$env:VALUES_FILE_PATH'"
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
