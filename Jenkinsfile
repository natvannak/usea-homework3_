pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)

        buildDiscarder(
            logRotator(
                numToKeepStr: '20',
                artifactNumToKeepStr: '10'
            )
        )
    }

    parameters {
        choice(
            name: 'ACTION',
            choices: [
                'build',
                'push',
                'deploy',
                'rollback'
            ],
            description: '''
                        build    = Build Docker image only
                        push     = Build and push image to AWS ECR
                        deploy   = Build, push and deploy to Docker Swarm
                        rollback = Roll back Docker Swarm service
                        '''
        )

        gitParameter(
            name: 'VERSION',
            type: 'PT_TAG',
            defaultValue: '',
            description: 'Select Git tag, for example v1.0.1',
            tagFilter: '*',
            sortMode: 'DESCENDING_SMART',
            selectedValue: 'TOP',
            quickFilterEnabled: true
        )
    }

    environment {
        /*
         * AWS ECR configuration
         */
        AWS_REGION      = 'us-east-1'
        AWS_ACCOUNT_ID  = '464604123652'
        ECR_REPO        = 'usea-homework3'

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_IMAGE    = "${ECR_REGISTRY}/${ECR_REPO}"

        /*
         * Docker Swarm Manager configuration
         */
        MANAGER_HOST = '54.82.194.125'
        MANAGER_USER = 'ubuntu'
        SERVICE_NAME = 'homework3_web'

        /*
         * Jenkins credential IDs
         */
        AWS_CREDENTIALS = 'aws-credentials'
        SSH_CREDENTIALS = 'swarm-ssh-key'

        /*
         * Values will be assigned during parameter validation
         */
        IMAGE_TAG      = ''
        FULL_IMAGE_NAME = ''

        /*
         * macOS Jenkins command paths
         */
        PATH = '/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin'
    }

    stages {
        stage('Validate Parameters') {
            steps {
                script {
                    def action = params.ACTION?.trim()
                    def version = params.VERSION?.trim()

                    echo '======================================'
                    echo 'Pipeline Parameters'
                    echo '======================================'
                    echo "ACTION  : ${action}"
                    echo "VERSION : ${version ?: '(not selected)'}"
                    echo '======================================'

                    if (!action) {
                        error('ACTION is required.')
                    }

                    if (action in ['build', 'push', 'deploy']) {
                        if (!version) {
                            error(
                                'VERSION is required. ' +
                                'Create and push a Git tag first.'
                            )
                        }

                        /*
                         * Docker tag validation
                         */
                        if (!(version ==~ /^[A-Za-z0-9][A-Za-z0-9._-]*$/)) {
                            error(
                                "Invalid VERSION '${version}'. " +
                                'Use letters, numbers, dots, underscores ' +
                                'or hyphens only.'
                            )
                        }

                        env.IMAGE_TAG = version
                        env.FULL_IMAGE_NAME =
                            "${env.ECR_IMAGE}:${version}"

                        echo "IMAGE_TAG       : ${env.IMAGE_TAG}"
                        echo "FULL_IMAGE_NAME : ${env.FULL_IMAGE_NAME}"
                    }

                    if (action in ['deploy', 'rollback']) {
                        if (!env.MANAGER_HOST?.trim()) {
                            error('MANAGER_HOST is empty.')
                        }

                        if (
                            env.MANAGER_HOST ==
                            'YOUR_SWARM_MANAGER_PUBLIC_IP'
                        ) {
                            error(
                                'Configure the real Docker Swarm ' +
                                'Manager IP address.'
                            )
                        }

                        if (!env.SERVICE_NAME?.trim()) {
                            error('SERVICE_NAME is empty.')
                        }
                    }
                }
            }
        }

        stage('Checkout Selected Git Tag') {
            when {
                expression {
                    params.ACTION in ['build', 'push', 'deploy']
                }
            }

            steps {
                checkout scm

                sh '''
                    set -eu

                    echo "======================================"
                    echo "Fetching Git tags"
                    echo "======================================"

                    git fetch --tags --force origin

                    echo "Selected tag: ${IMAGE_TAG}"

                    if ! git rev-parse \
                        -q \
                        --verify \
                        "refs/tags/${IMAGE_TAG}^{commit}" \
                        >/dev/null
                    then
                        echo "ERROR: Git tag does not exist:"
                        echo "${IMAGE_TAG}"
                        exit 1
                    fi

                    echo "Checking out tag: ${IMAGE_TAG}"

                    git checkout \
                        --force \
                        "refs/tags/${IMAGE_TAG}"

                    echo "======================================"
                    echo "Git Information"
                    echo "======================================"
                    echo "Tag    : ${IMAGE_TAG}"
                    echo "Commit : $(git rev-parse --short HEAD)"
                    echo "Branch : $(git branch --show-current || true)"
                    echo "======================================"
                '''
            }
        }

        stage('Check Required Commands') {
            when {
                expression {
                    params.ACTION in ['build', 'push', 'deploy']
                }
            }

            steps {
                sh '''
                    set -eu

                    echo "======================================"
                    echo "Checking required commands"
                    echo "======================================"

                    command -v git >/dev/null 2>&1 || {
                        echo "ERROR: Git is not installed."
                        exit 1
                    }

                    command -v docker >/dev/null 2>&1 || {
                        echo "ERROR: Docker is not installed."
                        exit 1
                    }

                    docker version

                    if [ "${ACTION}" = "push" ] ||
                       [ "${ACTION}" = "deploy" ]
                    then
                        command -v aws >/dev/null 2>&1 || {
                            echo "ERROR: AWS CLI is not installed."
                            exit 1
                        }

                        aws --version
                    fi
                '''
            }
        }

        stage('Build Docker Image') {
            when {
                expression {
                    params.ACTION in ['build', 'push', 'deploy']
                }
            }

            steps {
                sh '''
                    set -eu

                    echo "======================================"
                    echo "Building Docker image"
                    echo "======================================"
                    echo "Image: ${FULL_IMAGE_NAME}"

                    if [ ! -f Dockerfile ]; then
                        echo "ERROR: Dockerfile was not found."
                        exit 1
                    fi

                    docker build \
                        --pull \
                        --tag "${FULL_IMAGE_NAME}" \
                        .

                    echo "======================================"
                    echo "Docker image built successfully"
                    echo "======================================"

                    docker image inspect \
                        "${FULL_IMAGE_NAME}" \
                        --format \
                        'Image ID: {{.Id}}'

                    docker image inspect \
                        "${FULL_IMAGE_NAME}" \
                        --format \
                        'Created: {{.Created}}'
                '''
            }
        }

        stage('Push to AWS ECR') {
            when {
                expression {
                    params.ACTION in ['push', 'deploy']
                }
            }

            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {
                    sh '''
                        set -eu

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        echo "======================================"
                        echo "Checking AWS identity"
                        echo "======================================"

                        aws sts get-caller-identity

                        echo "======================================"
                        echo "Checking ECR repository"
                        echo "======================================"
                        echo "Region     : ${AWS_REGION}"
                        echo "Repository : ${ECR_REPO}"

                        aws ecr describe-repositories \
                            --region "${AWS_REGION}" \
                            --repository-names "${ECR_REPO}" \
                            >/dev/null

                        echo "ECR repository exists."

                        echo "======================================"
                        echo "Logging in to AWS ECR"
                        echo "======================================"
                        echo "Registry: ${ECR_REGISTRY}"

                        aws ecr get-login-password \
                            --region "${AWS_REGION}" \
                        | docker login \
                            --username AWS \
                            --password-stdin \
                            "${ECR_REGISTRY}"

                        echo "======================================"
                        echo "Checking local Docker image"
                        echo "======================================"

                        docker image inspect \
                            "${FULL_IMAGE_NAME}" \
                            >/dev/null

                        echo "======================================"
                        echo "Pushing Docker image"
                        echo "======================================"
                        echo "Image: ${FULL_IMAGE_NAME}"

                        docker push "${FULL_IMAGE_NAME}"

                        echo "======================================"
                        echo "Verifying image in AWS ECR"
                        echo "======================================"

                        aws ecr describe-images \
                            --region "${AWS_REGION}" \
                            --repository-name "${ECR_REPO}" \
                            --image-ids \
                            imageTag="${IMAGE_TAG}" \
                            --query \
                            'imageDetails[0].[imageTags,imageDigest,imagePushedAt]' \
                            --output table

                        echo "======================================"
                        echo "Push completed successfully"
                        echo "${FULL_IMAGE_NAME}"
                        echo "======================================"
                    '''
                }
            }
        }

        stage('Test SSH Connection') {
            when {
                anyOf {
                    expression {
                        params.ACTION == 'deploy'
                    }

                    expression {
                        params.ACTION == 'rollback'
                    }
                }
            }

            steps {
                sshagent(['swarm-ssh-key']) {
                    sh '''
                        set -eu

                        echo "======================================"
                        echo "Testing SSH connection"
                        echo "======================================"
                        echo "Server: ${MANAGER_USER}@${MANAGER_HOST}"

                        ssh \
                            -o BatchMode=yes \
                            -o ConnectTimeout=15 \
                            -o StrictHostKeyChecking=no \
                            -o UserKnownHostsFile=/dev/null \
                            "${MANAGER_USER}@${MANAGER_HOST}" \
                            '
                                set -eu

                                echo "Hostname : $(hostname)"
                                echo "User     : $(whoami)"

                                command -v docker >/dev/null 2>&1 || {
                                    echo "ERROR: Docker is not installed."
                                    exit 1
                                }

                                docker info >/dev/null

                                echo "SSH and Docker connection successful."
                            '
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
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {
                    sshagent(['swarm-ssh-key']) {
                        sh '''
                            set -eu

                            TOKEN_FILE="${WORKSPACE}/ecr-token.txt"
                            REMOTE_TOKEN="/tmp/homework3-ecr-token.txt"

                            cleanup_local() {
                                rm -f "${TOKEN_FILE}"
                            }

                            trap cleanup_local EXIT INT TERM

                            echo "======================================"
                            echo "Creating temporary ECR token"
                            echo "======================================"

                            aws ecr get-login-password \
                                --region "${AWS_REGION}" \
                                > "${TOKEN_FILE}"

                            chmod 600 "${TOKEN_FILE}"

                            echo "Copying ECR token to Swarm Manager..."

                            scp \
                                -o BatchMode=yes \
                                -o ConnectTimeout=15 \
                                -o StrictHostKeyChecking=no \
                                -o UserKnownHostsFile=/dev/null \
                                "${TOKEN_FILE}" \
                                "${MANAGER_USER}@${MANAGER_HOST}:${REMOTE_TOKEN}"

                            echo "======================================"
                            echo "Deploying to Docker Swarm"
                            echo "======================================"
                            echo "Service: ${SERVICE_NAME}"
                            echo "Image  : ${FULL_IMAGE_NAME}"

                            ssh \
                                -o BatchMode=yes \
                                -o ConnectTimeout=15 \
                                -o StrictHostKeyChecking=no \
                                -o UserKnownHostsFile=/dev/null \
                                "${MANAGER_USER}@${MANAGER_HOST}" \
                                "
                                    set -eu

                                    REMOTE_TOKEN='${REMOTE_TOKEN}'

                                    cleanup_remote() {
                                        rm -f \\"\${REMOTE_TOKEN}\\"
                                    }

                                    trap cleanup_remote EXIT INT TERM

                                    echo 'Logging Docker into AWS ECR...'

                                    cat \\"\${REMOTE_TOKEN}\\" |
                                    docker login \\
                                        --username AWS \\
                                        --password-stdin \\
                                        '${ECR_REGISTRY}'

                                    echo 'Checking Docker Swarm status...'

                                    SWARM_STATE=\$(docker info \\
                                        --format '{{.Swarm.LocalNodeState}}')

                                    echo \\"Swarm state: \${SWARM_STATE}\\"

                                    if [ \\"\${SWARM_STATE}\\" != 'active' ]
                                    then
                                        echo 'ERROR: Docker Swarm is not active.'
                                        exit 1
                                    fi

                                    echo 'Checking Docker service...'

                                    if ! docker service inspect \\
                                        '${SERVICE_NAME}' \\
                                        >/dev/null 2>&1
                                    then
                                        echo 'ERROR: Service was not found:'
                                        echo '${SERVICE_NAME}'
                                        echo
                                        echo 'Available services:'
                                        docker service ls
                                        exit 1
                                    fi

                                    echo 'Updating Docker Swarm service...'

                                    docker service update \\
                                        --with-registry-auth \\
                                        --image '${FULL_IMAGE_NAME}' \\
                                        --update-parallelism 1 \\
                                        --update-delay 10s \\
                                        --update-failure-action rollback \\
                                        --detach=false \\
                                        '${SERVICE_NAME}'

                                    echo 'Deployment completed successfully.'
                                "
                        '''
                    }
                }
            }
        }

        stage('Rollback Docker Swarm') {
            when {
                expression {
                    params.ACTION == 'rollback'
                }
            }

            steps {
                sshagent(['swarm-ssh-key']) {
                    sh '''
                        set -eu

                        echo "======================================"
                        echo "Rolling back Docker Swarm service"
                        echo "======================================"
                        echo "Service: ${SERVICE_NAME}"

                        ssh \
                            -o BatchMode=yes \
                            -o ConnectTimeout=15 \
                            -o StrictHostKeyChecking=no \
                            -o UserKnownHostsFile=/dev/null \
                            "${MANAGER_USER}@${MANAGER_HOST}" \
                            "
                                set -eu

                                if ! docker service inspect \\
                                    '${SERVICE_NAME}' \\
                                    >/dev/null 2>&1
                                then
                                    echo 'ERROR: Service was not found:'
                                    echo '${SERVICE_NAME}'
                                    echo
                                    echo 'Available services:'
                                    docker service ls
                                    exit 1
                                fi

                                echo 'Starting service rollback...'

                                docker service rollback \\
                                    --detach=false \\
                                    '${SERVICE_NAME}'

                                echo 'Rollback completed successfully.'
                            "
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            when {
                anyOf {
                    expression {
                        params.ACTION == 'deploy'
                    }

                    expression {
                        params.ACTION == 'rollback'
                    }
                }
            }

            steps {
                sshagent(['swarm-ssh-key']) {
                    sh '''
                        set -eu

                        echo "======================================"
                        echo "Verifying Docker Swarm deployment"
                        echo "======================================"

                        ssh \
                            -o BatchMode=yes \
                            -o ConnectTimeout=15 \
                            -o StrictHostKeyChecking=no \
                            -o UserKnownHostsFile=/dev/null \
                            "${MANAGER_USER}@${MANAGER_HOST}" \
                            "
                                set -eu

                                echo '======================================'
                                echo 'Docker Swarm Nodes'
                                echo '======================================'

                                docker node ls

                                echo '======================================'
                                echo 'Docker Swarm Service'
                                echo '======================================'

                                docker service ls \\
                                    --filter name='${SERVICE_NAME}'

                                echo '======================================'
                                echo 'Current Docker Image'
                                echo '======================================'

                                docker service inspect \\
                                    --format \
                                    '{{.Spec.TaskTemplate.ContainerSpec.Image}}' \\
                                    '${SERVICE_NAME}'

                                echo '======================================'
                                echo 'Service Replicas'
                                echo '======================================'

                                docker service inspect \\
                                    --format \
                                    '{{.Spec.Mode.Replicated.Replicas}}' \\
                                    '${SERVICE_NAME}'

                                echo '======================================'
                                echo 'Service Tasks'
                                echo '======================================'

                                docker service ps \\
                                    --no-trunc \\
                                    '${SERVICE_NAME}'
                            "
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '======================================'
            echo 'Pipeline completed successfully.'
            echo "ACTION  : ${params.ACTION}"
            echo "VERSION : ${params.VERSION ?: '(not required)'}"

            script {
                if (env.FULL_IMAGE_NAME?.trim()) {
                    echo "IMAGE   : ${env.FULL_IMAGE_NAME}"
                }
            }

            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'Pipeline failed.'
            echo 'Check the failed stage logs above.'
            echo '======================================'
        }

        aborted {
            echo '======================================'
            echo 'Pipeline was aborted.'
            echo '======================================'
        }

        always {
            sh '''
                rm -f \
                    "${WORKSPACE}/ecr-token.txt" \
                    2>/dev/null || true
            '''

            cleanWs(
                cleanWhenNotBuilt: false,
                deleteDirs: true,
                disableDeferredWipeout: false,
                notFailBuild: true
            )
        }
    }
}