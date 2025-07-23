pipeline {
    agent any
    options {
        timeout(time: 30, unit: 'MINUTES')  // Timeout for the entire pipeline
    }
    parameters {
        string(name: 'GIT_REPO', defaultValue: 'https://github.com/SrivenkateswaraReddy/k3s-argocd-deployments.git', description: 'Git repo with ArgoCD manifests')
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Git branch to checkout')
        string(name: 'ARGOCD_APP', defaultValue: 'metalb-app', description: 'ArgoCD application name to sync')
        string(name: 'ARGOCD_SERVER_PORT', defaultValue: '8005', description: 'Local port for ArgoCD port-forwarding')
        string(name: 'ARGOCD_USER', defaultValue: 'admin', description: 'ArgoCD username')
    }
    environment {
        ARGOCD_PASSWORD = credentials('argocd-password')  // Jenkins Credentials ID for ArgoCD admin password
        PATH = "/usr/local/bin:$PATH"  // Ensure system-wide path for argocd
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: params.GIT_BRANCH, url: params.GIT_REPO
            }
        }
        stage('Install ArgoCD CLI') {
            steps {
                script {
                    if (sh(script: 'command -v argocd', returnStatus: true) != 0) {
                        sh '''
                            echo "Downloading ArgoCD CLI for ARM64..."
                            curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-arm64
                            chmod +x argocd
                            sudo mv argocd /usr/local/bin/argocd
                        '''
                        sh 'argocd version || echo "ArgoCD CLI installation failed."'
                    } else {
                        echo 'ArgoCD CLI already installed.'
                    }
                }
            }
        }
        stage('Setup Port Forward & UFW') {
            steps {
                script {
                    def port = params.ARGOCD_SERVER_PORT
                    def ufwExists = sh(script: 'command -v ufw', returnStatus: true) == 0
                    def ufwActive = false
                    if (ufwExists) {
                        ufwActive = sh(script: 'sudo ufw status | grep -q "Status: active"', returnStatus: true) == 0
                    }

                    if (ufwExists && ufwActive) {
                        echo "UFW is active, allowing port ${port}"
                        sh "sudo ufw allow ${port}/tcp"
                        sh "sudo ufw reload"
                    } else {
                        echo "UFW not installed or inactive, skipping ufw allow"
                    }

                    echo "Starting kubectl port-forward on port ${port}..."
                    sh "nohup kubectl -n argocd port-forward svc/argocd-server ${port}:443 > port-forward.log 2>&1 &"
                    sleep(time: 5, unit: 'SECONDS')
                }
            }
        }
        stage('ArgoCD Login') {
            steps {
                script {
                    def server = "localhost:${params.ARGOCD_SERVER_PORT}"
                    echo "Logging into ArgoCD at ${server}..."
                    sh 'argocd version'  // Check again if installed
                    sh "argocd login ${server} --username ${params.ARGOCD_USER} --password '${env.ARGOCD_PASSWORD}' --insecure"
                }
            }
        }
        stage('Sync Application') {
            steps {
                script {
                    echo "Syncing ArgoCD app: ${params.ARGOCD_APP}"
                    sh "argocd app sync ${params.ARGOCD_APP}"
                }
            }
        }
    }
    post {
        always {
            echo "Cleaning up port-forward process..."
            sh '''
                pkill -f "kubectl -n argocd port-forward svc/argocd-server" || true
            '''
        }
    }
}
