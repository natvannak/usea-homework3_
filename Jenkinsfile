pipeline {

    agent any

    parameters {

        choice(
            name: 'ACTION',
            choices: [
                'build',
                'push',
                'deploy',
                'rollback'
            ],
            description: 'Pipeline action'
        )

        string(
            name: 'VERSION',
            defaultValue: '',
            description: 'Version such as v1.0.0'
        )
    }

    environment {

        AWS_REGION = 'us-east-1'

        AWS_ACCOUNT_ID = '464604123652'

        ECR_REPO = 'usea-homework3'

        ECR_REGISTRY =
            "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        ECR_IMAGE =
            "${ECR_REGISTRY}/${ECR_REPO}"

        MANAGER_HOST = '54.82.194.125'

        STACK_NAME = 'homework3'

        SERVICE_NAME = 'homework3_web'

        SSH_CREDENTIALS = 'swarm-ssh-key'
        VERSION = "${TAG_NAME}"

        PATH = '/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stages {
            stage('Check Version') {
                steps {
                    sh '''
                        if [ -z "$VERSION" ]; then
                            echo "VERSION is required"
                            exit 1
                        fi

                        echo "VERSION=$VERSION"
                    '''
                }
            }
        }
        

        stage('Build') {
            when {
                expression {
                    params.ACTION == 'build' ||
                    params.ACTION == 'push'
                }
            }

            steps {
                sh '''
                    docker build \
                      -t ${ECR_IMAGE}:${VERSION} .
                '''
            }
        }

        stage('Push') {
            when {
                expression {
                    params.ACTION == 'push'
                }
            }

            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials']
                ]) {

                    sh '''
                        aws ecr get-login-password \
                          --region ${AWS_REGION} \
                        | docker login \
                          --username AWS \
                          --password-stdin \
                          ${ECR_REGISTRY}

                        docker push \
                          ${ECR_IMAGE}:${VERSION}
                    '''
                }
            }
        }

        stage('Deploy') {
            when {
                expression {
                    params.ACTION == 'deploy'
                }
            }

            steps {
                sshagent(['swarm-ssh-key']) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                            ubuntu@${MANAGER_HOST} \
                            "docker service update \
                             --image ${ECR_IMAGE}:${VERSION} \
                             ${SERVICE_NAME}"
                    '''
                }
            }
        }

        stage('Rollback') {
            when {
                expression {
                    params.ACTION == 'rollback'
                }
            }

            steps {
                sshagent(['swarm-ssh-key']) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                            ubuntu@${MANAGER_HOST} \
                            "docker pull ${ECR_IMAGE}:${VERSION} && \
                             docker service update \
                             --image ${ECR_IMAGE}:${VERSION} \
                             ${SERVICE_NAME}"
                    '''
                }
            }
        }
    }
}