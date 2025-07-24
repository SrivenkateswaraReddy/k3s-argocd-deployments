pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
    }

    parameters {
        string(name: 'GIT_REPO',   defaultValue: 'https://github.com/SrivenkateswaraReddy/k3s-argocd-deployments.git',  description: 'Git repo with Argo CD manifests')
        string(name: 'GIT_BRANCH', defaultValue: 'main',                                                                description: 'Git branch to checkout')
        string(name: 'ARGOCD_APP', defaultValue: 'metalb-app',                                                           description: 'Argo CD application name to sync')
        string(name: 'ARGOCD_SERVER_PORT', defaultValue: '8005',                                                         description: 'Local port for Argo CD port-forwarding')
    }

    /* Keep secrets out of the global environment so the CLI can be called
       without triggering the Groovy-string–interpolation warning. */
    environment {
        PATH = "/usr/local/bin:$PATH"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: params.GIT_BRANCH, url: params.GIT_REPO
            }
        }

        stage('Install Argo CD CLI') {
            steps {
                script {
                    if (sh(script: 'command -v argocd', returnStatus: true) != 0) {
                        sh '''
                            echo "Downloading Argo CD CLI for ARM64..."
                            curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-arm64
                            chmod +x argocd
                            sudo mv argocd /usr/local/bin/argocd
                        '''
                        sh 'argocd version || echo "Argo CD CLI installation failed."'
                    } else {
                        echo 'Argo CD CLI already installed.'
                    }
                }
            }
        }

        stage('Setup port-forward & UFW') {
            steps {
                script {
                    def port = params.ARGOCD_SERVER_PORT.toInteger()
                    def ufwExists  = sh(script: 'command -v ufw', returnStatus: true) == 0
                    def ufwActive  = ufwExists && (sh(script: 'sudo ufw status | grep -q "Status: active"', returnStatus: true) == 0)

                    if (ufwActive) {
                        echo "UFW is active, allowing port ${port}"
                        sh "sudo ufw allow ${port}/tcp"
                        sh "sudo ufw reload"
                    } else {
                        echo "UFW not installed or inactive, skipping ufw allow"
                    }

                    echo "Starting kubectl port-forward on port ${port}…"
                    sh "nohup kubectl -n argocd port-forward svc/argocd-server ${port}:443 > port-forward.log 2>&1 &"
                    sleep 5
                }
            }
        }

        stage('Argo CD login (token)') {
            steps {
                /* Pull the token only for the commands that need it. */
                withCredentials([string(credentialsId: 'argocd-token', variable: 'ARGO_TOKEN')]) {
                    script {
                        def server = "localhost:${params.ARGOCD_SERVER_PORT}"
                        echo "Logging into Argo CD at ${server} using token…"
                        sh """
                            #!/usr/bin/env bash
                            set -e
                            # \$ARGO_TOKEN is provided by withCredentials and masked by Jenkins.
                            argocd login ${server} --grpc-web --auth-token \$ARGO_TOKEN --insecure
                        """
                    }
                }
            }
        }

        stage('Sync application') {
            steps {
                echo "Syncing Argo CD app: ${params.ARGOCD_APP}"
                sh "argocd app sync ${params.ARGOCD_APP}"
            }
        }
    }

    post {
        always {
            echo 'Cleaning up port-forward process…'
            sh 'pkill -f "kubectl -n argocd port-forward svc/argocd-server" || true'
        }
    }
}
