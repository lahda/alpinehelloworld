/* import shared library */
@Library('shared-library')_

pipeline {
    agent none

    environment {
        DOCKERHUB_AUTH = credentials('dockerhub-creds')
        ID_DOCKER      = "${DOCKERHUB_AUTH_USR}"
        IMAGE_NAME     = "alpinehelloworld"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        CONTAINER_TEST = "alpinehelloworld-${BUILD_NUMBER}"
        TEST_PORT      = "5001"
        STAGING_HOST   = "100.48.94.15"
        PROD_HOST      = "54.234.164.41"
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
      script {
        slackNotifier currentBuild.result
      }
    }  
  }
}
