pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        CLUSTER_NAME = "bluegreen-eks"
        NAMESPACE = "production"

        DOCKER_IMAGE = "yourdockerhubuser/sample-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                url: 'https://github.com/your-org/sample-app.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh """
                    echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin

                    docker push ${DOCKER_IMAGE}:${IMAGE_TAG}

                    docker logout
                    """
                }
            }
        }

        stage('Create EKS Cluster') {
            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds']
                ]) {

                    sh """
                    eksctl get cluster \
                    --name ${CLUSTER_NAME} \
                    --region ${AWS_REGION} || \

                    eksctl create cluster \
                    --name ${CLUSTER_NAME} \
                    --region ${AWS_REGION} \
                    --nodegroup-name workers \
                    --node-type t3.medium \
                    --nodes 2 \
                    --managed
                    """
                }
            }
        }

        stage('Configure kubectl') {
            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds']
                ]) {

                    sh """
                    aws eks update-kubeconfig \
                    --region ${AWS_REGION} \
                    --name ${CLUSTER_NAME}

                    kubectl get nodes
                    """
                }
            }
        }

        stage('Create Namespace') {
            steps {
                sh """
                kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                """
            }
        }

        stage('Deploy Green Version') {
            steps {

                sh """
cat <<EOF | kubectl apply -f -

apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
  namespace: ${NAMESPACE}

spec:
  replicas: 2

  selector:
    matchLabels:
      app: sample-app
      version: green

  template:
    metadata:
      labels:
        app: sample-app
        version: green

    spec:
      containers:
      - name: sample-app
        image: ${DOCKER_IMAGE}:${IMAGE_TAG}

        ports:
        - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: sample-service
  namespace: ${NAMESPACE}

spec:
  selector:
    app: sample-app
    version: green

  ports:
  - port: 80
    targetPort: 8080

EOF
                """
            }
        }

        stage('Validate Deployment') {
            steps {

                sh """
                kubectl rollout status deployment/app-green -n ${NAMESPACE}

                kubectl get pods -n ${NAMESPACE}

                kubectl get svc -n ${NAMESPACE}
                """
            }
        }

        stage('Delete Blue Deployment') {
            steps {

                sh """
                kubectl delete deployment app-blue \
                -n ${NAMESPACE} \
                --ignore-not-found=true
                """
            }
        }
    }

    post {

        success {
            echo "Blue-Green deployment completed successfully"
        }

        failure {
            echo "Deployment failed"
        }
    }
}
