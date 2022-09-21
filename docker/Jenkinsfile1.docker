pipeline {
    agent {
        label 'docker'
    }

    environment {
        PROJECT_NAME = "${env.JOB_NAME}-${env.BUILD_ID}"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr:'10'))
        disableConcurrentBuilds()
    }

    stages {
        /*
        Use Advanced Sub-Module Behaviors in Jenkins
        stage("Checkout") {
            steps {
                checkout scm
                sh 'git submodule update --init'
                stash(name:'scripts', includes:'**')
            }
        }
        */
        stage('Prepare') {
            steps {
                timeout(time:5, unit:'MINUTES') {
                    sh "docker-compose -p ${PROJECT_NAME} up -d testserver"
                }
            }
        }
        stage('Build') {
            agent {
                docker {
                    image 'registry.docker.tests:5000/softwareag/abe:10.1'
                    args '-v ${env.WORKSPACE}:/workspace'
                }
            }
            steps {
                timeout(time:1, unit:'MINUTES') {
                    sh "ant -DprojectName=${PROJECT_NAME} build"
                }
            }
        }
        stage('Deploy') {
            agent {
                docker {
                    image 'registry.docker.tests:5000/softwareag/deployer:10.1'
                    args '-v ${env.WORKSPACE}:/workspace'
                }
            }

            steps {
                timeout(time:10, unit:'MINUTES') {
                    sh "ant -DprojectName=${PROJECT_NAME} deploy"
                }
            }
        }
        stage('Tests') {
            agent {
                docker {
                    image 'registry.docker.tests:5000/softwareag/wmtestsuite:10.1'
                    args '-v ${env.WORKSPACE}:/workspace'
                }
            }
            steps {
                timeout(time:10, unit:'MINUTES') {
                    sh "ant -DprojectName=${PROJECT_NAME} test"
                }
            }
            post {
                always {
                    junit "report/*.xml"
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