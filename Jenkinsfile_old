def skipRemainingStages = false

pipeline {
    //this should be given by jenkins at build time
    environment {
        PROJECT_NAME = "${env.JOB_NAME}"
        
        BUILD_NAME = "${PROJECT_NAME}"
        BUILD_VERSION = "0.0.${env.BUILD_ID}"
        BUILD_PROFILE = "prod"
        BUILD_FINAL = "${BUILD_NAME}-${BUILD_VERSION}".toLowerCase()
        BUILD_LATEST = "${BUILD_NAME}-latest".toLowerCase()
        PACKAGE_S3_BUCKET_REGION = "us-east-1"
        PACKAGE_S3_BUCKET = "sedemos-prod-main"
        PACKAGE_S3_BUCKET_PREFIX = "cicd_builds/${PROJECT_NAME}"
    }

    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr:'10'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Build') {
            steps {
                echo "Building code for ${PROJECT_NAME} with profile ${BUILD_PROFILE}"
                timeout(time:1, unit:'MINUTES') {
                    sh "${env.SAG_HOME}/common/lib/ant/bin/ant -DSAGHome=${env.SAG_HOME} -DSAG_CI_HOME=${env.SAG_CI_HOME} -DprojectName=${PROJECT_NAME} build"
                }
            }
        }
        
        stage('Deploy to Test Server') {
            steps {
				sh "${env.SAG_HOME}/common/lib/ant/bin/ant -DSAGHome=${env.SAG_HOME} -DSAG_CI_HOME=${env.SAG_CI_HOME} -DprojectName=${PROJECT_NAME} deploy"
            }
        }
        
	 	stage('Unit Tests') {
	        steps {
				sh "${env.SAG_HOME}/common/lib/ant/bin/ant -DSAGHome=${env.SAG_HOME} -DSAG_CI_HOME=${env.SAG_CI_HOME} -DprojectName=${PROJECT_NAME} test"
				
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
	    
        stage('Final Packaging') {
            when {
                expression {
                    !skipRemainingStages
                }
            }
            
            steps {
                echo "Packaging build - ${BUILD_FINAL}"

                fileOperations(
                    [
                    	folderCopyOperation(sourceFolderPath: "./target/${PROJECT_NAME}/build/", destinationFolderPath: "./packaging/${BUILD_FINAL}/"),
                        fileZipOperation(folderPath: "./packaging/${BUILD_FINAL}/", outputFolderPath: "./packaging/"),
                        folderDeleteOperation("./packaging/${BUILD_FINAL}/")
                    ]
                )
            }
        }
        
        stage('Push to S3 Repo') {
            when {
                expression {
                    !skipRemainingStages
                }
            }
            
            steps {
	            echo "Upload final build to s3"
	            withAWS(region:"${PACKAGE_S3_BUCKET_REGION}") {
	                s3Upload(
	                    file:"./packaging/${BUILD_FINAL}.zip", 
	                    bucket:"${PACKAGE_S3_BUCKET}", 
	                    path:"${PACKAGE_S3_BUCKET_PREFIX}/${BUILD_FINAL}.zip",
	                    pathStyleAccessEnabled: true, 
	                    payloadSigningEnabled: true,
	                )
	                
	                s3Upload(
	                    file:"./packaging/${BUILD_FINAL}.zip",
	                    bucket:"${PACKAGE_S3_BUCKET}", 
	                    path:"${PACKAGE_S3_BUCKET_PREFIX}/${BUILD_LATEST}.txt",
	                    text:"s3://${PACKAGE_S3_BUCKET}/${PACKAGE_S3_BUCKET_PREFIX}/${BUILD_FINAL}.zip",
	                    pathStyleAccessEnabled: true, 
	                    payloadSigningEnabled: true,
	                )
	            }
            }
        }
    }
         
    post {
        success {
        	echo "success...final tasks..."
            echo "Cleaning up"
            fileOperations(
                [
                    fileDeleteOperation(excludes: "", includes: "./packaging/${BUILD_FINAL}.zip")
                ]
            )
        }
		unstable {
	        echo "Unstable - Upload status to S3"
	        withAWS(region:"${PACKAGE_S3_BUCKET_REGION}") {
	        	s3Upload(
		           file:"/dev/null",
		           bucket:"${PACKAGE_S3_BUCKET}", 
		           path:"${PACKAGE_S3_BUCKET_PREFIX}/${BUILD_FINAL}-unstable",
		           text:"Unstable build",
		           pathStyleAccessEnabled: true, 
		           payloadSigningEnabled: true,
	        	)
	        }
		}
		
		failure {
	        echo "Failed note - Upload status to S3"
	        withAWS(region:"${PACKAGE_S3_BUCKET_REGION}") {
	        	s3Upload(
		           file:"/dev/null",
		           bucket:"${PACKAGE_S3_BUCKET}", 
		           path:"${PACKAGE_S3_BUCKET_PREFIX}/${BUILD_FINAL}-failed",
		           text:"Failed build",
		           pathStyleAccessEnabled: true, 
		           payloadSigningEnabled: true,
	        	)
	        }
		}
    }
}