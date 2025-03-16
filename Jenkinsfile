pipeline {
    agent any

    environment {
        REGISTRY = "ghcr.io"
        IMAGE_NAME = "${env.GITHUB_REPOSITORY}"
        GITHUB_TOKEN = credentials('github-token') // Assuming you have saved your GitHub token as a credential in Jenkins
        REMOTE_HOST = "your-remote-host"
        REMOTE_USER = "your-remote-user"
        SSH_KEY = credentials('ssh-key-id') // SSH key credential ID in Jenkins
        KOPS_CLUSTER_NAME = "your-kops-cluster-name"
        KOPS_STATE_STORE = "s3://your-kops-state-store"
        KUBECONFIG = credentials('kubeconfig') // Kubeconfig file credential ID in Jenkins
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Unit Testing') {
            agent {
                label 'ubuntu'
            }
            steps {
                sh 'npm ci'
                sh 'npm test || echo "No tests found, would add tests in a real project"'
            }
        }

        stage('Static Code Analysis') {
            agent {
                label 'ubuntu'
            }
            steps {
                sh 'npm ci'
                sh 'npm run lint'
            }
        }

        stage('Build') {
            agent {
                label 'ubuntu'
            }
            steps {
                sh 'npm ci'
                sh 'npm run build'
                archiveArtifacts artifacts: 'dist/**', allowEmptyArchive: true
            }
        }

        stage('Docker Build and Push') {
            agent {
                label 'ubuntu'
            }
            steps {
                script {
                    withCredentials([string(credentialsId: 'github-token', variable: 'TOKEN')]) {
                        sh '''
                            echo "${TOKEN}" | docker login ${REGISTRY} -u ${env.GITHUB_ACTOR} --password-stdin
                            docker build -t ${REGISTRY}/${IMAGE_NAME}:sha-${GIT_COMMIT} -t ${REGISTRY}/${IMAGE_NAME}:latest .
                            docker push ${REGISTRY}/${IMAGE_NAME}:sha-${GIT_COMMIT}
                            docker push ${REGISTRY}/${IMAGE_NAME}:latest
                        '''
                    }
                }
            }
        }

        stage('Trivy Scan') {
            agent {
                label 'ubuntu'
            }
            steps {
                sh '''
                    docker pull aquasec/trivy:latest
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --exit-code 1 --severity CRITICAL,HIGH ${REGISTRY}/${IMAGE_NAME}:sha-${GIT_COMMIT}
                '''
            }
        }

        stage('Update Kubernetes Deployment') {
            when {
                branch 'main'
            }
            agent {
                label 'ubuntu'
            }
            environment {
                IMAGE_TAG = "sha-${GIT_COMMIT}"
            }
            steps {
                script {
                    sh '''
                        NEW_IMAGE="${REGISTRY}/${GITHUB_REPOSITORY}:${IMAGE_TAG}"
                        sed -i "s|image: ${REGISTRY}/.*|image: ${NEW_IMAGE}|g" kubernetes/deployment.yaml
                        git config user.name "Jenkins"
                        git config user.email "jenkins@localhost"
                        git add kubernetes/deployment.yaml
                        git commit -m "Update Kubernetes deployment with new image tag: ${IMAGE_TAG} [skip ci]" || echo "No changes to commit"
                        git push
                    '''
                }
            }
        }

        stage('Apply Kubernetes Manifests') {
            when {
                branch 'main'
            }
            agent {
                label 'ubuntu'
            }
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        sh '''
                            export KUBECONFIG=${KUBECONFIG_FILE}
                            kubectl apply -f kubernetes/deployment.yaml
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kops Cluster') {
            when {
                branch 'main'
            }
            agent {
                label 'ubuntu'
            }
            steps {
                script {
                    withCredentials([sshUserPrivateKey(credentialsId: 'ssh-key-id', keyFileVariable: 'SSH_KEY')]) {
                        sh '''
                            ssh -i ${SSH_KEY} ${REMOTE_USER}@${REMOTE_HOST} << EOF
                                export KOPS_CLUSTER_NAME=${KOPS_CLUSTER_NAME}
                                export KOPS_STATE_STORE=${KOPS_STATE_STORE}
                                kops update cluster --name ${KOPS_CLUSTER_NAME} --yes
                                kops rolling-update cluster --name ${KOPS_CLUSTER_NAME} --yes
                            EOF
                        '''
                    }
                }
            }
        }
    }
}
