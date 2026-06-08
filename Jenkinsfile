pipeline {
    agent none

    environment {
        DOCKERHUB_AUTH = credentials('id_dockerhub')
        ID_DOCKER      = "${DOCKERHUB_AUTH_USR}"
        IMAGE_NAME     = "alpinehelloworld"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        CONTAINER_TEST = "alpinehelloworld-${BUILD_NUMBER}"
        TEST_PORT      = "5001"
        STAGING_HOST   = "100.54.199.53"
        PROD_HOST      = "100.30.187.194"
    }

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {

        stage('Build Image') {
            agent any
            steps {
                sh 'docker build -t ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }

        stage('Run container') {
            agent any
            steps {
                sh '''
                    echo "Preparing environment..."
                    
                    # Force clean any lingering container occupying our test port (5001)
                    OLD_CONTAINER=$(docker ps -q -f "publish=${TEST_PORT}")
                    if [ ! -z "$OLD_CONTAINER" ]; then
                        echo "Found legacy container (${OLD_CONTAINER}) on port ${TEST_PORT}. Evicting..."
                        docker rm -f $OLD_CONTAINER
                    fi

                    # Ensure specific container name is vacant
                    docker rm -f ${CONTAINER_TEST} || true
                    
                    # Launch fresh test environment
                    docker run --name ${CONTAINER_TEST} -d \
                        -p ${TEST_PORT}:5000 \
                        -e PORT=5000 \
                        ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}
                    sleep 5
                '''
            }
        }

        stage('Test image') {
            agent any
            steps {
                sh '''
                    docker ps | grep ${CONTAINER_TEST} || (echo "Container failed to start!" && exit 1)
                    curl -f http://172.17.0.1:${TEST_PORT}/ | grep -q "Hello world!"
                    echo "Local integration test passed safely!"
                '''
            }
        }

        stage('Push Image to DockerHub') {
            agent any
            steps {
                sh '''
                    echo $DOCKERHUB_AUTH_PSW | docker login -u $DOCKERHUB_AUTH_USR --password-stdin
                    docker push ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker tag ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} ${ID_DOCKER}/${IMAGE_NAME}:latest
                    docker push ${ID_DOCKER}/${IMAGE_NAME}:latest
                    docker logout
                '''
            }
        }

        stage('Clean local test artifacts') {
            agent any
            steps {
                sh '''
                    docker rm -f ${CONTAINER_TEST} || true
                    docker rmi ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} || true
                '''
            }
        }

        stage('Deploy in staging') {
            agent any
            steps {
                sshagent(credentials: ['staging-ssh']) {
                    sh '''
                        command1="echo $DOCKERHUB_AUTH_PSW | docker login -u $DOCKERHUB_AUTH_USR --password-stdin"
                        command2="docker pull $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG"
                        command3="docker rm -f webapp || echo 'No staging app to replace'"
                        command4="docker run -d -p 80:5000 -e PORT=5000 --name webapp --restart always $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG"
                        command5="sleep 3 && docker ps | grep webapp"
                        ssh -o StrictHostKeyChecking=no ubuntu@${STAGING_HOST} "$command1 && $command2 && $command3 && $command4 && $command5"
                    '''
                }
            }
        }

        stage('Verify Staging') {
            agent any
            steps {
                sh '''
                    sleep 5
                    curl -f http://${STAGING_HOST}/ | grep -q "Hello world!"
                    echo "Staging environment is healthy!"
                '''
            }
        }

        stage('Approval for Production') {
            agent none
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: 'Staging validation passed. Deploy to PRODUCTION?',
                          ok: 'Oui, déployer !'
                }
            }
        }

        stage('Deploy in prod') {
            agent any
            steps {
                sshagent(credentials: ['prod-ssh']) {
                    sh '''
                        command1="echo $DOCKERHUB_AUTH_PSW | docker login -u $DOCKERHUB_AUTH_USR --password-stdin"
                        command2="docker pull $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG"
                        command3="docker rm -f webapp || echo 'No production app to replace'"
                        command4="docker run -d -p 80:5000 -e PORT=5000 --name webapp --restart always $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG"
                        command5="sleep 3 && docker ps | grep webapp"
                        ssh -o StrictHostKeyChecking=no ubuntu@${PROD_HOST} "$command1 && $command2 && $command3 && $command4 && $command5"
                    '''
                }
            }
        }

        stage('Verify Production') {
            agent any
            steps {
                sh '''
                    sleep 5
                    curl -f http://${PROD_HOST}/ | grep -q "Hello world!"
                    echo "Production live deployment verified successfully!"
                '''
            }
        }
    }

    post {
        always {
            // Correct declarative way to specify an agent context inside post-actions
            agent any
            steps {
                sh 'docker image prune -f || true'
            }
        }
        success {
            slackSend(
                color: 'good',
                message: """✅ *${env.JOB_NAME}* #${env.BUILD_NUMBER} réussi
- Image: `${env.ID_DOCKER}/${env.IMAGE_NAME}:${env.IMAGE_TAG}`
- Durée: ${currentBuild.durationString}
- <${env.BUILD_URL}|Voir le build>"""
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: """❌ *${env.JOB_NAME}* #${env.BUILD_NUMBER} échoué
- Stage: `${env.STAGE_NAME}`
- <${env.BUILD_URL}console|Voir les logs>"""
            )
        }
        aborted {
            slackSend(
                color: 'warning',
                message: """⚠️ *${env.JOB_NAME}* #${env.BUILD_NUMBER} annulé
- <${env.BUILD_URL}|Voir le build>"""
            )
        }
    }
}
