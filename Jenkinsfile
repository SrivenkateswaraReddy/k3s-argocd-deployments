pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
    }

    parameters {
        string(name: 'GIT_REPO',        defaultValue: 'https://github.com/SrivenkateswaraReddy/k3s-argocd-deployments.git', description: 'Git repo with Argo CD manifests')
        string(name: 'GIT_BRANCH',      defaultValue: 'main',                                                         description: 'Git branch to checkout')
        string(name: 'ARGOCD_APP',      defaultValue: 'metalb-app',                                                   description: 'Argo CD application name to sync')
        string(name: 'ARGOCD_SERVER_IP',defaultValue: '192.168.1.190',                                               description: 'Node IP address where Argo CD server NodePort is exposed')
        string(name: 'ARGOCD_SERVER_PORT', defaultValue: '30115',                                                   description: 'NodePort for Argo CD server')
    }

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

        /*
        // If you want to keep port-forwarding instead of using NodePort, uncomment this
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
        */

        stage('Argo CD login (username/password)') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'argocd-jenkins-creds', usernameVariable: 'ARGOCD_USER', passwordVariable: 'ARGOCD_PASSWORD')]) {
                    script {
                        def server = "${params.ARGOCD_SERVER_IP}:${params.ARGOCD_SERVER_PORT}"
                        echo "Logging into Argo CD at ${server} using username and password..."
                        sh """
                            set -e
                            argocd login ${server} --username \$ARGOCD_USER --password \$ARGOCD_PASSWORD --insecure
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
