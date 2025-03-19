pipeline {
    agent any
    
    environment {
        AWS_REGION = 'us-west-2'
        ECR_REPOSITORY_NAME = 'ecommerce-app'
        AWS_CREDENTIALS = 'my-aws-credentials'
        EKS_CLUSTER_NAME = 'my-eks-cluster'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        TERRAFORM_DIR = 'TerraformDep'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup Infrastructure') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
                                   credentialsId: "${AWS_CREDENTIALS}",
                                   accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
                                   secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    dir("${TERRAFORM_DIR}") {
                        sh '''
                            terraform init
                            terraform apply -auto-approve
                            aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}
                        '''
                    }
                }
            }
        }
        
        stage('Build and Push Image') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
                                   credentialsId: "${AWS_CREDENTIALS}",
                                   accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
                                   secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    sh '''
                        # Get AWS Account ID
                        AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
                        ECR_REPOSITORY_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY_NAME}"
                        
                        # Check if ECR repository exists; if not, create it
                        if ! aws ecr describe-repositories --repository-names ${ECR_REPOSITORY_NAME} --region ${AWS_REGION} > /dev/null 2>&1; then
                            aws ecr create-repository --repository-name ${ECR_REPOSITORY_NAME} --region ${AWS_REGION}
                        fi
                        
                        # Build image
                        docker build -t ${ECR_REPOSITORY_URI}:${IMAGE_TAG} -f webapp/Dockerfile webapp
                        
                        # Login to ECR and push
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPOSITORY_URI}
                        docker push ${ECR_REPOSITORY_URI}:${IMAGE_TAG}
                        
                        # Save image URI for later use
                        echo "${ECR_REPOSITORY_URI}:${IMAGE_TAG}" > image_uri.txt
                    '''
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
                                  credentialsId: "${AWS_CREDENTIALS}",
                                  accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
                                  secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    sh '''
                        # Get image URI
                        IMAGE_URI=$(cat image_uri.txt)
                        
                        # Create or update deployment
                        kubectl get deployment ecommerce-app || \
                        kubectl create deployment ecommerce-app --image=${IMAGE_URI}
                        
                        # If deployment exists, update the image
                        kubectl set image deployment/ecommerce-app ecommerce-app=${IMAGE_URI}
                        
                        # Create or update service
                        kubectl get service ecommerce-app || \
                        kubectl expose deployment ecommerce-app --type=LoadBalancer --port=80 --target-port=80
                        
                        # Wait for deployment to complete
                        kubectl rollout status deployment/ecommerce-app
                        
                        # Get the service URL
                        echo "Application URL: $(kubectl get service ecommerce-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
                    '''
                }
            }
        }
        stage('Destroy Kubernetes Resources') {
            steps {
                script {
                    // Delete the deployment
                    sh 'kubectl delete deployment ecommerce-app --ignore-not-found=true'
                    
                    // Delete the service
                    sh 'kubectl delete service ecommerce-app --ignore-not-found=true'
                    
                    // Optionally, delete other resources (e.g., ConfigMaps, Secrets)
                    sh 'kubectl delete configmap my-config --ignore-not-found=true'
                    sh 'kubectl delete secret my-secret --ignore-not-found=true'
                }
            }
        }
    }
    
    post {
        failure {
            echo 'Pipeline failed! Check the logs for details.'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
    }
}
