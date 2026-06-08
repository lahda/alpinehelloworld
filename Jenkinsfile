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
                    echo "Clean environment"
                    docker rm -f ${CONTAINER_TEST} || echo "container does not exist"
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
                    docker ps | grep ${CONTAINER_TEST} || (echo "Container not running!" && exit 1)
                    curl -f http://172.17.0.1:${TEST_PORT}/ | grep -q "Hello world!"
                    echo "Test passed!"
                '''
            }
        }

        stage('Clean container') {
            agent any
            steps {
                sh 'docker rm -f ${CONTAINER_TEST} || true'
            }
        }

        stage('Push Image to DockerHub') {
            agent any
            steps {
                sh '''
                    echo $DOCKERHUB_AUTH_PSW | docker login \
                        -u $DOCKERHUB_AUTH_USR \
                        --password-stdin
                    docker push ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker tag ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} \
                               ${ID_DOCKER}/${IMAGE_NAME}:latest
                    docker push ${ID_DOCKER}/${IMAGE_NAME}:latest
                    docker logout
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
                        command3="docker rm -f webapp || echo 'app does not exist'"
                        command4="docker run -d -p 80:5000 -e PORT=5000 --name webapp --restart always $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG"
                        command5="sleep 3 && docker ps | grep webapp"
                        ssh -o StrictHostKeyChecking=no ubuntu@${STAGING_HOST} \
                            "$command1 && $command2 && $command3 && $command4 && $command5"
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
                    echo "Staging is healthy!"
                '''
            }
        }

        stage('Approval for Production') {
            agent none
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: 'Staging OK ? Déployer en PRODUCTION ?',
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
                        command3="docker rm -f webapp || echo 'app does not exist'"
                        command4="docker run -d -p 80:5000 -e PORT=5000 --name webapp --restart always $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG"
                        command5="sleep 3 && docker ps | grep webapp"
                        ssh -o StrictHostKeyChecking=no ubuntu@${PROD_HOST} \
                            "$command1 && $command2 && $command3 && $command4 && $command5"
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
                    echo "Production is healthy!"
                '''
            }
        }
    }

    post {
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
