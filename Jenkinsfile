pipeline {
  agent any
  environment {
    DOCKER_HUB_REPO = "manjudata/manjuproject9"
    DOCKER_HUB_CREDENTIALS_ID = "gitops-docker-token"
    KUBECONFIG_CRED = "kubeconfig-file"   // <-- Jenkins credential: a real kubeconfig file
    PATH = "${WORKSPACE}/bin:${PATH}"
    NO_PROXY = "127.0.0.1,localhost,.svc,192.168.49.2"  // add API server host here if needed
  }

  stages {
    stage('Checkout Github') {
      steps {
        echo 'Checking out code from GitHub...'
        checkout scmGit(
          branches: [[name: '*/main']],
          extensions: [],
          userRemoteConfigs: [[credentialsId: 'github-token', url: 'https://github.com/manjudata/MLOPS-PROJECT-9.git']]
        )
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
          docker.withRegistry('https://registry.hub.docker.com', "${DOCKER_HUB_CREDENTIALS_ID}") {
            dockerImage.push('latest')
          }
        }
      }
    }

    stage('Install Kubectl & ArgoCD CLI Setup') {
      steps {
        sh '''
          set -e
          echo "Installing kubectl & argocd..."
          mkdir -p "$WORKSPACE/bin"
          curl -L -o "$WORKSPACE/bin/kubectl" "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
          chmod +x "$WORKSPACE/bin/kubectl"
          curl -sSL -o "$WORKSPACE/bin/argocd" https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
          chmod +x "$WORKSPACE/bin/argocd"

          # Client-only checks (no cluster contact, avoids prompts)
          kubectl version --client
          argocd version --client
        '''
      }
    }

    stage('Apply Kubernetes & Sync App with ArgoCD') {
      steps {
        script {
          // Use a real kubeconfig file credential; avoids any interactive prompts.
          withKubeConfig([credentialsId: env.KUBECONFIG_CRED]) {
            sh '''
              set -e
              echo "Sanity check cluster access..."
              kubectl auth can-i get pods --all-namespaces

              echo "Fetching Argo CD initial admin password..."
              ARGO_PASS=$(kubectl -n argocd get secret argocd-initial-admin-secret \
                           -o jsonpath="{.data.password}" | base64 -d)

              echo "Logging into Argo CD..."
              # Use --grpc-web for NodePort/ingress setups; keep --insecure if you don't have TLS certs
              argocd login 35.184.177.230:30097 --username admin --password "$ARGO_PASS" --insecure --grpc-web

              echo "Syncing app..."
              argocd app sync gitopsappnew
              argocd app wait gitopsappnew --health --sync
            '''
          }
        }
      }
    }
  }
}