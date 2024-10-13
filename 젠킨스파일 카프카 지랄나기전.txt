pipeline {
    agent any

    environment {
        KUBECONFIG = "C:\\Windows\\system32\\config\\systemprofile\\.kube\\config"
        REGION = 'ap-northeast-2'
        AWS_CREDENTIAL_NAME = 'aws-key'
        JSON_TEMPLATE_PATH = 'E:\\docker_Logi\\infra_structure\\policy-document-template.json' // 템플릿 파일 경로
        JSON_FILE_PATH = 'E:\\docker_Logi\\infra_structure\\policy-document.json' // JSON 파일 경로
    }

    stages {
        // Terraform으로 EKS 클러스터 및 자원 생성
        stage('Terraform Apply') {
            steps {
                dir('E:/docker_dev/terraform-codes') {
                    script {
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                            bat '''
                            set AWS_ACCESS_KEY_ID=%AWS_ACCESS_KEY_ID%
                            set AWS_SECRET_ACCESS_KEY=%AWS_SECRET_ACCESS_KEY%
                            terraform apply -auto-approve
                            '''
                        }
                    }
                }
            }
        }

        // Update IAM Role Trust Policy
        stage('Update IAM Role Trust Policy') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                        // OIDC 공급자를 먼저 가져온 후
                        def oidcProvider = powershell(script: 'aws eks describe-cluster --name test-eks-cluster --region ap-northeast-2 --query "cluster.identity.oidc.issuer" --output text', returnStdout: true).trim().replace('https://', '')
                        echo "oidcProvider: ${oidcProvider}"        

                        // policy-document-template.json을 복사하여 policy-document.json으로 저장
                        bat """
                        powershell -Command "Copy-Item -Path '${env.JSON_TEMPLATE_PATH}' -Destination '${env.JSON_FILE_PATH}'"
                        """

                        // PowerShell 스크립트를 사용하여 JSON 파일의 내용을 동적으로 업데이트
                        bat """
                        powershell -Command "(Get-Content '${env.JSON_FILE_PATH}' -Raw) -replace 'here', '${oidcProvider}' | Set-Content '${env.JSON_FILE_PATH}'"
                        """

                        // 업데이트된 JSON 파일 출력 (디버깅용)
                        bat """
                        powershell -Command "(Get-Content '${env.JSON_FILE_PATH}')"
                        """

                        // IAM 역할 신뢰 정책 업데이트
                        bat """
                        powershell -Command "aws iam update-assume-role-policy --role-name AmazonEKSEBSCSIRole --policy-document file://${env.JSON_FILE_PATH}"
                        """
                    }
                }
            }
        }

        // Install EBS CSI Driver
        stage('Install EBS CSI Driver') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                        bat '''
                        set KUBECONFIG=C:\\Windows\\system32\\config\\systemprofile\\.kube\\config
                        aws eks update-kubeconfig --region ap-northeast-2 --name test-eks-cluster
                        eksctl utils associate-iam-oidc-provider --region=ap-northeast-2 --cluster=test-eks-cluster --approve
                        eksctl get cluster --region ap-northeast-2
                        type C:\\Windows\\system32\\config\\systemprofile\\.kube\\config
                        kubectl get nodes
                        
                        kubectl apply -f E:/docker_Logi/infra_structure/ebs-csi-service-account.yaml
                        eksctl create addon --name aws-ebs-csi-driver --cluster test-eks-cluster --service-account-role-arn arn:aws:iam::339713037008:role/AmazonEKSEBSCSIRole --force --region ap-northeast-2
                        '''
                    }
                }
            }
        }

        // Apply Storage Class
        stage('Apply Storage Class') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                        bat '''
                        kubectl apply -f E:/docker_Logi/infra_structure/storage-class.yaml
                        '''
                    }
                }
            }
        }

        // Apply Kafka PVC
        stage('Apply Kafka PVC') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                        bat '''
                        kubectl apply -f E:/docker_Logi/infra_structure/kafka-pvc0.yaml
		kubectl apply -f E:/docker_Logi/infra_structure/kafka-pvc1.yaml
		kubectl apply -f E:/docker_Logi/infra_structure/kafka-pvc2.yaml
                        '''
                    }
                }
            }
        }

        // Install Helm Chart for Zookeeper and Kafka
        stage('Install Zookeeper and Kafka with Helm') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                        bat '''
                        set KUBECONFIG=C:\\Windows\\system32\\config\\systemprofile\\.kube\\config
                        helm repo add bitnami https://charts.bitnami.com/bitnami
                        helm repo update
                        
                        REM Install Kafka (with corrected settings)
                        helm install kafka bitnami/kafka -f E:/docker_Logi/infra_structure/values.yaml

                        bat "powershell -Command \"Start-Sleep -Seconds 120\""

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
            echo 'Infrastructure successfully deployed and PVC created!'
        }
        failure {
            echo 'Deployment failed.'
        }
    }
}
