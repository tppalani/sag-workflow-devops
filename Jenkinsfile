def skipRemainingStages = false

pipeline {

    parameters {
        string(description: 'Build stage env', name: 'BUILD_PROFILE')
        string(description: 'The deployer target alias', name: 'deployerTargetAlias')
        string(description: 'The host of the deployer server', name: 'deployerTargetHost')
        string(description: 'The port of the deployer server', name: 'deployerTargetPort')
        string(description: 'The credentials of the deployer server', name: 'deployerCredentialsID')
        string(description: 'The credentials of the target deployment server', name: 'deployerTargetCredentialsID')

        string(description: 'The host of the test server', name: 'testISHost')
        string(description: 'The port of the test server', name: 'testISPort')
        string(description: 'The credentials of the test server', name: 'testISCredentialsID')
    }

    environment {
        PROJECT_NAME = "${env.JOB_NAME}".toLowerCase()
        
        //Project name properties required for builds/deployment + params defined in the jenkins job
        projectNameCI = "${PROJECT_NAME}ci"
        projectNameCD = "${PROJECT_NAME}cd"
        
        BUILD_NAME = "${PROJECT_NAME}".toLowerCase()
        BUILD_VERSION = "0.0.${env.BUILD_ID}"
        BUILD_FINAL = "${BUILD_NAME}-${BUILD_PROFILE}-${BUILD_VERSION}".toLowerCase()
        BUILD_LATEST = "${BUILD_NAME}-latest".toLowerCase()

        //These are passed from environment variables
        ARCHIVE_S3_BUCKET_REGION = "${env.BUILDS_ARCHIVE_S3_BUCKET_REGION}"
        ARCHIVE_S3_BUCKET_NAME = "${env.BUILDS_ARCHIVE_S3_BUCKET}"
        ARCHIVE_S3_BUCKET_PATH = "${env.BUILDS_ARCHIVE_S3_BUCKET_PREFIX}/${BUILD_NAME}"
    }

    agent { label 'webmethods' }

   
    options {
        buildDiscarder(logRotator(numToKeepStr:'10'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Build') {
            steps {
                echo "Building code for project ${projectNameCI} with profile ${BUILD_PROFILE}"
                timeout(time:1, unit:'MINUTES') {
                    sh "ant -DprojectName=${projectNameCI} build"
                }
            }
        }
        
        stage('Deploy to Test Server') {
            steps {
                withCredentials(
                    [
                        usernamePassword(credentialsId: "${deployerCredentialsID}", passwordVariable: 'deployerPassword', usernameVariable: 'deployerUsername'), 
                        usernamePassword(credentialsId: "${testISCredentialsID}", passwordVariable: 'testISPassword', usernameVariable: 'testISUsername')
                    ]
                ) {
                    sh "ant -DprojectName=${projectNameCI} deploy2test"
                }
            }
        }
        
        stage('Unit Tests') {
	        steps {
                withCredentials(
                    [
                        usernamePassword(credentialsId: "${testISCredentialsID}", passwordVariable: 'testISPassword', usernameVariable: 'testISUsername')
                    ]
                ) {
                    sh "ant -DprojectName=${projectNameCI} test"
                }

                // no tests for now..
                junit 'report/'
				
				script {
					//do not execute the remaining steps if junit tests failed
					if (currentBuild.result == 'UNSTABLE'){
						skipRemainingStages = true
					}
					
					println "skipRemainingStages = ${skipRemainingStages}"
                }
	        }
	    }
	    
        stage('Package And Archive') {
            when {
                expression {
                    !skipRemainingStages
                }
            }
            
            steps {
                echo "Packaging build - ${BUILD_FINAL}"
                fileOperations(
                    [
                    	folderCreateOperation('./packaging'),
                    	folderCopyOperation(sourceFolderPath: "./target/${projectNameCI}/build/", destinationFolderPath: "./packaging/${BUILD_FINAL}/"),
                        fileZipOperation(folderPath: "./packaging/${BUILD_FINAL}/", outputFolderPath: "./packaging/"),
                    ]
                )

	            echo "Upload final build to s3"
	            withAWS(region:"${ARCHIVE_S3_BUCKET_REGION}") {
	                s3Upload(
	                    file:"./packaging/${BUILD_FINAL}.zip", 
	                    bucket:"${ARCHIVE_S3_BUCKET_NAME}", 
	                    path:"${ARCHIVE_S3_BUCKET_PATH}/${BUILD_FINAL}.zip",
	                    pathStyleAccessEnabled: true, 
	                    payloadSigningEnabled: true,
	                )
	                
	                s3Upload(
	                    file:"./packaging/${BUILD_FINAL}.zip",
	                    bucket:"${ARCHIVE_S3_BUCKET_NAME}", 
	                    path:"${ARCHIVE_S3_BUCKET_PATH}/${BUILD_LATEST}.txt",
	                    text:"s3://${ARCHIVE_S3_BUCKET_NAME}/${ARCHIVE_S3_BUCKET_PATH}/${BUILD_FINAL}.zip",
	                    pathStyleAccessEnabled: true, 
	                    payloadSigningEnabled: true,
	                )
	            }

                echo "Cleaning up packaging workspace"
                fileOperations(
                    [
                        folderDeleteOperation("./packaging"),
                        folderDeleteOperation("./target")
                    ]
                )
            }
        }
        
        stage('Deploy To Target') {
            when {
                expression {
                    !skipRemainingStages
                }
            }
            
            steps {
                echo "Creating deployment workspace"
                fileOperations(
                    [
                        folderCreateOperation('./deployment')
                    ]
                )
                
                echo "Downloading final build from s3"
	            withAWS(region:"${ARCHIVE_S3_BUCKET_REGION}") {
	                s3Download(
	                    file:"./deployment/${BUILD_FINAL}.zip", 
	                    bucket:"${ARCHIVE_S3_BUCKET_NAME}", 
	                    path:"${ARCHIVE_S3_BUCKET_PATH}/${BUILD_FINAL}.zip",
	                    pathStyleAccessEnabled: true, 
	                    payloadSigningEnabled: true,
	                )
	            }

				echo "UnPacking build - ${BUILD_FINAL}"
                fileOperations(
                    [
                        fileUnZipOperation(filePath: "./deployment/${BUILD_FINAL}.zip", targetLocation: "./deployment")
                    ]
                )
                
                echo "Deploying - ${BUILD_FINAL} - to target"
                withCredentials(
                    [
                        usernamePassword(credentialsId: "${deployerCredentialsID}", passwordVariable: 'deployerPassword', usernameVariable: 'deployerUsername'), 
                        usernamePassword(credentialsId: "${deployerTargetCredentialsID}", passwordVariable: 'deployerTargetPassword', usernameVariable: 'deployerTargetUsername')
                    ]
                ) {
                    sh "ant -DprojectName=${projectNameCD} -DrepositoryPath=${WORKSPACE}/deployment/${BUILD_FINAL} deploy2target"
                }

                echo "Cleaning up deployment workspace"
                fileOperations(
                    [
                        folderDeleteOperation("./deployment")
                    ]
                )
            }
        }

        stage('Post Deployment Tests') {
	        when {
                expression {
                    !skipRemainingStages
                }
            }
            
            steps {
	            echo "Executing deployment validation tests...TBD"
	        }
	    }
    }
         
    post {
        success {
        	echo "success...final tasks..."
            echo "Cleaning up workspace"
        }
		unstable {
	        echo "Unstable - Upload status to S3"
	        withAWS(region:"${ARCHIVE_S3_BUCKET_REGION}") {
	        	s3Upload(
		           file:"/dev/null",
		           bucket:"${ARCHIVE_S3_BUCKET_NAME}", 
		           path:"${ARCHIVE_S3_BUCKET_PATH}/${BUILD_FINAL}-unstable",
		           text:"Unstable build",
		           pathStyleAccessEnabled: true, 
		           payloadSigningEnabled: true,
	        	)
	        }
		}
		
		failure {
	        echo "Failed note - Upload status to S3"
	        withAWS(region:"${ARCHIVE_S3_BUCKET_REGION}") {
	        	s3Upload(
		           file:"/dev/null",
		           bucket:"${ARCHIVE_S3_BUCKET_NAME}", 
		           path:"${ARCHIVE_S3_BUCKET_PATH}/${BUILD_FINAL}-failed",
		           text:"Failed build",
		           pathStyleAccessEnabled: true, 
		           payloadSigningEnabled: true,
	        	)
	        }
		}
    }
}
