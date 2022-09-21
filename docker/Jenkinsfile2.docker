pipeline {
    agent {
        label 'docker'
    }

    environment {
        PROJECT_NAME = "${env.JOB_NAME}-${env.BUILD_ID}"
        /*WORKSPACE = "/var/jenkins_home/workspace/${env.JOB_NAME}"*/
        REGISTRY = "registry.docker.tests:5000"
        IMAGE_NAME = "${env.JOB_NAME}".toLowerCase()
        IMAGE_VERSION = "10.1-${env.BUILD_ID}"
        IMAGE_PREFIX = "softwareag"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr:'10'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Prepare Tests') {
            steps {
                timeout(time:5, unit:'MINUTES') {
                    sh "docker-compose -p ${PROJECT_NAME} up -d testserver"
                }
            }
        }
        stage('Build') {
            steps {
                timeout(time:1, unit:'MINUTES') {
                    sh "docker-compose -p ${PROJECT_NAME} up compile"
                }
            }
        }
        stage('Deploy for Tests') {
            steps {
                timeout(time:10, unit:'MINUTES') {
                    sh 'docker-compose -p ${PROJECT_NAME} up deploy'
                }
            }
        }
        stage('Run Tests') {
            steps {
                timeout(time:10, unit:'MINUTES') {
                    sh 'docker-compose -p ${PROJECT_NAME} up unittests'
                }
            }
            post {
                always {
                    junit "report/*.xml"
                }
            }
        }
        stage('Save to Docker Registry') {
            steps {
                echo 'Saving Docker image'
                script {
                    docker.withRegistry('https://${REGISTRY}') {
                        def customImage = docker.build("${IMAGE_PREFIX}/${IMAGE_NAME}:${IMAGE_VERSION}")
                        customImage.push()
                        customImage.push("latest")
                    }
                }
            }
        }
    }

    post {
        always {
            sh "docker-compose -p ${PROJECT_NAME} down -v || true"
        }
    }
}