pipeline {
    agent any
    parameters {
        choice(name: 'BRANCH', choices: ['master', 'dev'], description: 'Выбери ветку')
        choice(name: 'SERVER', choices: ['vps-kube', 'local-kube'], description: 'Выбери сервер')
        string(name: 'REPO_OWNER', defaultValue: 'arch-hcra', description: 'Владелец')
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: "${params.BRANCH}", url: "https://github.com/${params.REPO_OWNER}/ast_jav_tst.git", credentialsId: 'git-token'
            }
        }
        stage('Login to GHCR') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'git-token', variable: 'GHCR_TOKEN')]) {
                        sh "echo \$GHCR_TOKEN | docker login ghcr.io -u ${params.REPO_OWNER} --password-stdin"
                    }
                }
            }
        }
        stage('Build and Push Image') {
            steps {
                script {
                    def IMAGE_TAG = "${params.BRANCH}-${env.GIT_COMMIT.take(7)}" 
                    sh "docker build -t ghcr.io/${params.REPO_OWNER}/ast_jav_tst/app_j:${IMAGE_TAG} ."
                    sh "docker push ghcr.io/${params.REPO_OWNER}/ast_jav_tst/app_j:${IMAGE_TAG}"
                    env.IMAGE_TAG = IMAGE_TAG 
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    def KUBE_CONFIG_ID = params.SERVER == 'vps-kube' ? 'kubeconfig-vps' : 'kubeconfig-local'
                    withCredentials([file(credentialsId: KUBE_CONFIG_ID, variable: 'KUBECONFIG')]) {
                        sh """
                            export KUBECONFIG=\$KUBECONFIG
                            DEPLOYMENT="myapp"
                            kubectl set image deployment/\$DEPLOYMENT app_j=ghcr.io/${params.REPO_OWNER}/ast_jav_tst/app_j:\$IMAGE_TAG -n default
                            kubectl rollout restart deployment/\$DEPLOYMENT -n default
                            kubectl rollout status deployment/\$DEPLOYMENT -n default --timeout=300s
                        """
                    }
                }
            }
        }
    }
    post {
        success {
            echo 'Деплой успешен! Pod обновлён.'
        }
        failure {
            echo 'Ошибка в pipeline. Проверь логи.'
        }
    }
}

