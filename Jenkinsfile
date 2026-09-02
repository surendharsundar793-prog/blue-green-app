pipeline {
    agent any

    environment {
        AWS_REGION   = 'ap-south-1'
        CLUSTER_NAME = 'eks-cluster'
        DOCKER_IMAGE = 'surendharr/blue-green-app'
        IMAGE_TAG    = "${BUILD_NUMBER}"
        SWITCHED     = 'false'
    }

    stages {

        stage('Git checkout') {
            steps {
                git 'https://github.com/surendharsundar793-prog/blue-green-app.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Tag Docker Image') {
            steps {
                sh '''
                    docker tag ${DOCKER_IMAGE}:${IMAGE_TAG} \
                    ${DOCKER_IMAGE}:latest
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh '''
                            docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                            docker push ${DOCKER_IMAGE}:latest
                        '''
                    }
                }
            }
        }

        stage('Detect Active Environment') {
            steps {
                withAWS(
                    credentials: 'aws-cred',
                    region: "${AWS_REGION}"
                ) {
                    script {

                        sh '''
                            aws eks update-kubeconfig \
                                --region ${AWS_REGION} \
                                --name ${CLUSTER_NAME}
                        '''

                        env.ACTIVE_ENV = sh(
                            script: '''
                                kubectl get svc blue-green-service \
                                -o jsonpath='{.spec.selector.version}'
                            ''',
                            returnStdout: true
                        ).trim()

                        if (env.ACTIVE_ENV == 'blue') {
                            env.TARGET_ENV = 'green'
                            env.TARGET_DEPLOYMENT = 'green-deployment'
                            env.CONTAINER_NAME = 'green-app'
                        } else {
                            env.TARGET_ENV = 'blue'
                            env.TARGET_DEPLOYMENT = 'blue-deployment'
                            env.CONTAINER_NAME = 'blue-green-app'
                        }

                        echo "======================================"
                        echo "Current LIVE environment : ${env.ACTIVE_ENV}"
                        echo "Target environment       : ${env.TARGET_ENV}"
                        echo "Target deployment        : ${env.TARGET_DEPLOYMENT}"
                        echo "Container name           : ${env.CONTAINER_NAME}"
                        echo "======================================"
                    }
                }
            }
        }

        stage('Deploy Inactive Environment') {
            steps {
                withAWS(
                    credentials: 'aws-cred',
                    region: "${AWS_REGION}"
                ) {
                    sh '''
                        kubectl apply \
                            -f k8s/${TARGET_ENV}-deployment.yaml

                        kubectl set image \
                            deployment/${TARGET_DEPLOYMENT} \
                            ${CONTAINER_NAME}=${DOCKER_IMAGE}:${IMAGE_TAG}

                        kubectl rollout status \
                            deployment/${TARGET_DEPLOYMENT} \
                            --timeout=180s
                    '''
                }
            }
        }

        stage('Validate New Environment') {
            steps {
                withAWS(
                    credentials: 'aws-cred',
                    region: "${AWS_REGION}"
                ) {
                    sh '''
                        echo "Validating ${TARGET_ENV} environment..."

                        kubectl get deployment \
                            ${TARGET_DEPLOYMENT}

                        kubectl get pods \
                            -l app=blue-green-app,version=${TARGET_ENV}

                        kubectl wait \
                            --for=condition=available \
                            deployment/${TARGET_DEPLOYMENT} \
                            --timeout=180s

                        echo "${TARGET_ENV} environment is healthy."
                    '''
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                withAWS(
                    credentials: 'aws-cred',
                    region: "${AWS_REGION}"
                ) {
                    sh '''
                        kubectl patch service blue-green-service \
                            -p '{"spec":{"selector":{"app":"blue-green-app","version":"'"${TARGET_ENV}"'"}}}'

                        echo "Traffic switched from ${ACTIVE_ENV} to ${TARGET_ENV}"

                        kubectl get service blue-green-service \
                            -o jsonpath='{.spec.selector}'
                    '''

                    script {
                        env.SWITCHED = 'true'
                    }
                }
            }
        }

        stage('Verify Traffic Switch') {
            steps {
                withAWS(
                    credentials: 'aws-cred',
                    region: "${AWS_REGION}"
                ) {
                    sh '''
                        CURRENT_ENV=$(kubectl get service blue-green-service \
                            -o jsonpath='{.spec.selector.version}')

                        echo "Expected environment : ${TARGET_ENV}"
                        echo "Actual environment   : ${CURRENT_ENV}"

                        test "$CURRENT_ENV" = "${TARGET_ENV}"

                        echo "Traffic successfully verified on ${TARGET_ENV}"
                    '''
                }
            }
        }
    }

    post {
    success {
        echo '======================================'
        echo 'BLUE-GREEN DEPLOYMENT SUCCESSFUL'
        echo "LIVE ENVIRONMENT: ${env.TARGET_ENV}"
        echo '======================================'
    }

    failure {
        echo '======================================'
        echo 'BLUE-GREEN DEPLOYMENT FAILED'
        echo 'PREVIOUS ENVIRONMENT REMAINS AVAILABLE'
        echo '======================================'
        }
    }
}
