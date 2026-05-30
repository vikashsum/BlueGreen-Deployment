pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        CLUSTER_NAME = 'bluegreen-eks'
        NAMESPACE = 'production'

        DOCKER_IMAGE = 'vikash3117/sample-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

       stage('Checkout') {
    steps {
        deleteDir()
        git branch: 'main',
            url: 'https://github.com/vikashsum/BlueGreen-Deployment.git'
    }
}

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                """
            }
        }

        stage('Docker Login') {
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
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh """
                docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                """
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
                kubectl create namespace ${NAMESPACE} \
                --dry-run=client -o yaml | kubectl apply -f -
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

        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5

        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10

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

  type: LoadBalancer

EOF
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                   kubectl rollout status deployment/app-green -n production --timeout=300s
                     '''

                kubectl get pods -n ${NAMESPACE}

                kubectl get svc -n ${NAMESPACE}
                """
            }
        }
    }

    post {
        success {
            echo 'Blue-Green deployment successful'
        }

        failure {
            echo 'Pipeline failed'
        }
    }
}
