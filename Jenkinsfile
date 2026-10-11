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
            description: 'Version such as v1, v2, v3'
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

        PATH =
            "/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Version') {
            steps {
                script {
                    if (!params.VERSION?.trim()) {
                        error("VERSION parameter is required")
                    }

                    echo "Version: ${params.VERSION}"
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
                sh """
                    docker build \
                    -t ${ECR_IMAGE}:${params.VERSION} .
                """
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

                    sh """
                        aws ecr get-login-password \
                        --region ${AWS_REGION} \
                        | docker login \
                        --username AWS \
                        --password-stdin \
                        ${ECR_REGISTRY}

                        docker push \
                        ${ECR_IMAGE}:${params.VERSION}
                    """
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

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials']
                ]) {

                    sshagent(['swarm-ssh-key']) {

                        sh """
                        ssh -o StrictHostKeyChecking=no \
                        ubuntu@${MANAGER_HOST} '

                        aws ecr get-login-password \
                        --region ${AWS_REGION} \
                        | docker login \
                        --username AWS \
                        --password-stdin \
                        ${ECR_REGISTRY}

                        docker service update \
                        --with-registry-auth \
                        --image ${ECR_IMAGE}:${params.VERSION} \
                        ${SERVICE_NAME}
                        '
                        """
                    }
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

                    sh """
                    ssh -o StrictHostKeyChecking=no \
                    ubuntu@${MANAGER_HOST} '

                    docker service rollback \
                    ${SERVICE_NAME}

                    '
                    """
                }
            }
        }
    }

    post {

        success {
            echo "Pipeline completed successfully"
        }

        failure {
            echo "Pipeline failed"
        }

        always {
            cleanWs()
        }
    }
}