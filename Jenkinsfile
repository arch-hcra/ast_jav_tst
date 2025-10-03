pipeline {
    agent any

    options {
        buildDiscarder(logRotator(daysToKeepStr: '7'))
    }
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
                        sh "echo \$GHCR_TOKEN | docker login ghcr.io -u ${params.REPO_OWNER} --password-stdin 2>&1 | tee login_log.txt"
                    }
                }
            }
        }
        stage('Build and Push Image') {
            steps {
                script {
                    def IMAGE_TAG = "${params.BRANCH}-${env.GIT_COMMIT.take(7)}"
                    sh """
                        docker build -t ghcr.io/${params.REPO_OWNER}/ast_jav_tst/app_j:${IMAGE_TAG} . 2>&1 | tee build_push_log.txt
                        docker push ghcr.io/${params.REPO_OWNER}/ast_jav_tst/app_j:${IMAGE_TAG} 2>&1 | tee -a build_push_log.txt
                    """
                    env.IMAGE_TAG = IMAGE_TAG
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                script {
                    sh """
                        echo "Branch: ${params.BRANCH}" > build-info.txt
                        echo "Commit: ${env.GIT_COMMIT}" >> build-info.txt
                        echo "Image Tag: ${env.IMAGE_TAG}" >> build-info.txt
                        echo "Server: ${params.SERVER}" >> build-info.txt
                        echo "Build Time: \$(date)" >> build-info.txt
                    """
                  
                    archiveArtifacts artifacts: 'build-info.txt, login_log.txt, build_push_log.txt, deploy_log.txt', allowEmptyArchive: true
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
                            DEPLOYMENT="appj"
                            kubectl set image deployment/\$DEPLOYMENT appj=ghcr.io/${params.REPO_OWNER}/ast_jav_tst/app_j:\$IMAGE_TAG -n default 2>&1 | tee deploy_log.txt
                            kubectl rollout restart deployment/\$DEPLOYMENT -n default 2>&1 | tee -a deploy_log.txt
                            kubectl rollout status deployment/\$DEPLOYMENT -n default --timeout=300s 2>&1 | tee -a deploy_log.txt
                        """
                    }
                }
            }
        }

        stage('Build Docker Image with Artifacts') {
            steps {
                
                sh 'mkdir -p docker-context && cp build-info.txt build_push_log.txt docker-context/'
                
               
                sh 'docker build -t ${DOCKER_IMAGE} docker-context'
                
                
                sh 'docker push ${DOCKER_IMAGE}'
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

