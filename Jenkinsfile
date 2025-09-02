pipeline {
    agent any
    environment {
        DOCKER_HUB_REPO = "manjudata/manjuproject9"
        DOCKER_HUB_CREDENTIALS_ID = "gitops-docker-token"
    }
    stages {
        stage('Checkout Github') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'github-token', url: 'https://github.com/manjudata/MLOPS-PROJECT-9.git']])
                    }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image...'
                    dockerImage = docker.build("${DOCKER_HUB_REPO}:latest")
                }
            }
        }    
        stage('Push Image to DockerHub') {
            steps {
                script {
                    echo 'Pushing Docker image to DockerHub...'
                    docker.withRegistry('https://registry.hub.docker.com' , "${DOCKER_HUB_CREDENTIALS_ID}") {
                        dockerImage.push('latest')
                    }
                }
            }
        }
        stage('Install Kubectl & ArgoCD CLI Setup') {
            steps {
                sh '''
                echo 'installing Kubectl & ArgoCD cli...'
                curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                chmod +x kubectl
                mv kubectl /usr/local/bin/kubectl
                curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
                chmod +x /usr/local/bin/argocd
                '''
            }
        }


        stage('Check Argo token credential') {
            steps {
            withCredentials([string(credentialsId: 'argocd-token', variable: 'ARGO_TOKEN')]) {
            sh 'echo "ARGO token available"'
             }
           }
        }

        stage('Apply Kubernetes & Sync App with ArgoCD') {
            steps {
                withCredentials([string(credentialsId: 'argocd-token', variable: 'ARGO_TOKEN')]) {
                sh '''
                set -euo pipefail
                # login via token (no prompts); grpc-web/insecure for NodePort without TLS
                argocd login 35.184.177.230:30097 --auth-token "5TlCjsQqjtHfn-JR" --insecure --grpc-web

                # sync and wait until healthy
                argocd app sync gitopsappnew
                argocd app wait gitopsappnew --health --sync
            '''
            }
           }
        }
    }
        
}
