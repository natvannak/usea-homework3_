
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    parameters {
        choice(
            name: 'ACTION',
            choices: ['build', 'push', 'deploy', 'rollback'],
            description: 'Select pipeline action'
        )

        gitParameter(
            name: 'VERSION',
            type: 'PT_TAG',
            defaultValue: '',
            description: 'Select Git Tag, e.g. v1.0.1',
            tagFilter: '*',
            sortMode: 'DESCENDING_SMART',
            selectedValue: 'TOP'
        )
    }

    environment {
        AWS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = '464604123652'
        ECR_REPO       = 'usea-homework3'

        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_IMAGE      = "${ECR_REGISTRY}/${ECR_REPO}"

        // Replace with your Swarm Manager's reachable public IP or DNS.
        MANAGER_HOST   = '54.82.194.125'
        SERVICE_NAME   = 'homework3_web'

        IMAGE_TAG      = ''
        PATH = '/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin'

    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm

                sh '''
                    set -eu
                    git fetch --tags --force origin
                    echo "Current commit:"
                    git rev-parse --short HEAD
                    echo "Available tags:"
                    git tag --list
                '''
            }
        }

        stage('Validate Parameters') {
            steps {
                script {
                    def version = params.VERSION?.trim() ?: ''

                    echo "ACTION  : ${params.ACTION}"
                    echo "VERSION : ${version ?: '(not selected)'}"

                    if (params.ACTION in ['build', 'push', 'deploy']) {
                        if (!version) {
                            error(
                                'VERSION is required. ' +
                                'Create and push a Git tag, then select it.'
                            )
                        }

                        // Confirm the selected tag exists locally.
                        def tagExists = sh(
                            script: "git rev-parse -q --verify 'refs/tags/${version}'",
                            returnStatus: true
                        ) == 0

                        if (!tagExists) {
                            error("Git tag '${version}' does not exist.")
                        }
                    }
                }
            }
        }

        stage('Build') {
            when {
                expression {
                    params.ACTION in ['build', 'push']
                }
            }

            steps {
                script {

                    def version = params.VERSION.trim()

                    sh """
                        docker build \
                            --pull \
                            -t ${ECR_IMAGE}:${version} \
                            .
                    """
                }
            }
            
        }

        stage('Push to AWS ECR') {
            when {
                expression {
                    params.ACTION == 'push'
                }
            }

            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    sh '''
                        set -eu

                        aws sts get-caller-identity

                        aws ecr describe-repositories \
                            --region "$AWS_REGION" \
                            --repository-names "$ECR_REPO" \
                            >/dev/null

                        aws ecr get-login-password \
                            --region "$AWS_REGION" \
                        | docker login \
                            --username AWS \
                            --password-stdin "$ECR_REGISTRY"

                        docker push "${ECR_IMAGE}:${IMAGE_TAG}"

                        echo "Successfully pushed:"
                        echo "${ECR_IMAGE}:${IMAGE_TAG}"
                    '''
                }
            }
        }

        stage('Deploy to Docker Swarm') {
            when {
                expression {
                    params.ACTION == 'deploy'
                }
            }

            steps {
                script {
                    def version = params.VERSION.trim()

                    if (env.MANAGER_HOST == 'YOUR_SWARM_MANAGER_PUBLIC_IP') {
                        error('Configure MANAGER_HOST with your Swarm Manager IP.')
                    }

                    withCredentials([
                        [
                            $class: 'AmazonWebServicesCredentialsBinding',
                            credentialsId: 'aws-credentials'
                        ]
                    ]) {
                        sshagent(['swarm-ssh-key']) {
                            sh """
                                set -eu

                                aws ecr get-login-password \\
                                    --region '${AWS_REGION}' > ecr-token.txt

                                chmod 600 ecr-token.txt

                                scp -o StrictHostKeyChecking=no \\
                                    ecr-token.txt \\
                                    ubuntu@${MANAGER_HOST}:/tmp/ecr-token.txt

                                ssh -o StrictHostKeyChecking=no \\
                                    ubuntu@${MANAGER_HOST} '
                                        set -eu

                                        trap "rm -f /tmp/ecr-token.txt" EXIT

                                        cat /tmp/ecr-token.txt | docker login \\
                                            --username AWS \\
                                            --password-stdin \\
                                            ${ECR_REGISTRY}

                                        docker service inspect ${SERVICE_NAME} >/dev/null

                                        docker service update \\
                                            --with-registry-auth \\
                                            --image ${ECR_IMAGE}:${version} \\
                                            --detach=false \\
                                            ${SERVICE_NAME}

                                        echo "Deployment completed."
                                    '

                                rm -f ecr-token.txt
                            """
                        }
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
                script {
                    if (env.MANAGER_HOST == 'YOUR_SWARM_MANAGER_PUBLIC_IP') {
                        error('Configure MANAGER_HOST with your Swarm Manager IP.')
                    }

                    sshagent(['swarm-ssh-key']) {
                        sh """
                            set -eu

                            ssh -o StrictHostKeyChecking=no \\
                                ubuntu@${MANAGER_HOST} '
                                    set -eu

                                    docker service inspect ${SERVICE_NAME} >/dev/null

                                    docker service rollback \\
                                        --detach=false \\
                                        ${SERVICE_NAME}

                                    echo "Rollback completed."
                                '
                        """
                    }
                }
            }
        }

        stage('Verify Deployment') {
            when {
                anyOf {
                    expression { params.ACTION == 'deploy' }
                    expression { params.ACTION == 'rollback' }
                }
            }

            steps {
                sshagent(['swarm-ssh-key']) {
                    sh """
                        set -eu

                        ssh -o StrictHostKeyChecking=no \\
                            ubuntu@${MANAGER_HOST} '
                                echo "=== Swarm Service ==="
                                docker service ls

                                echo "=== Service Details ==="
                                docker service inspect \\
                                    --format "{{.Spec.TaskTemplate.ContainerSpec.Image}}" \\
                                    ${SERVICE_NAME}

                                echo "=== Service Tasks ==="
                                docker service ps \\
                                    --no-trunc \\
                                    ${SERVICE_NAME}
                            '
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the stage logs above.'
        }

        always {
            sh 'rm -f ecr-token.txt || true'
            cleanWs()
        }
    }
}
