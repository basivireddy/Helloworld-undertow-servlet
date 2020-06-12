podTemplate(
label: "lgi-jenkins-slave-maven",
cloud: "openshift",
inheritFrom: "maven",
containers: [
        containerTemplate(
                name: "jnlp",
                image: "docker-registry.default.svc:5000/cicd/jenkins-slave-maven",
                resourceRequestMemory: "1Gi",
                resourceLimitMemory: "2Gi",
                )
        ]
)




{
    def SERVICE_NAME = "helloworld"
	def BUILD_IMAGE = "openjdk18-openshift"
    def DEV_PROJECT = "test"
	def CPU_REQUESTS = "250m"
    def CPU_LIMITS = "500m"
	def MEM_REQUESTS = "512Mi"
	def MEM_LIMITS = "1Gi"

    node('jenkins-slave-maven') {

        // Checkout Source Code
         stage('Checkout Source') {
			 //source is in same repo as Jenkinsfile
             checkout scm
         }

        // Define Maven Command. Make sure it points to the correct settings for our Nexus installation
        // The file nexus_settings.xml needs to be in the Source Code repository.
        def mvnCmd = "mvn -s maven-settings.xml"
        echo "mvnCmd: ${mvnCmd}"

        dir('.') {

            // The following variables need to be defined at the top level
            // and not inside the scope of a stage - otherwise they would not be accessible from other stages.
            // Extract version and other properties from the pom.xml
            def groupId    = getGroupIdFromPom("pom.xml")
            echo "groupId: ${groupId}"
            def artifactId = getArtifactIdFromPom("pom.xml")
            echo "artifactId: ${artifactId}"
            def version    = getVersionFromPom("pom.xml")
            def pomVersion    = "${version}"
            echo "pomVersion: ${pomVersion}"

            GIT_COMMIT_HASH = sh (script: "git log -n 1 --pretty=format:'%H'", returnStdout: true)
            GIT_COMMIT_HASH = GIT_COMMIT_HASH.toString().substring(0,7)
            def commitId = "${GIT_COMMIT_HASH}"
            echo "commitId: ${commitId}"

            if (getVersionFromPom("pom.xml").toString().toUpperCase().contains("SNAPSHOT")) {

                version = getVersionFromPom("pom.xml").toString().toUpperCase().replace("-SNAPSHOT", "")

            } else {
                //If the condition is false print the following statement
                version = getVersionFromPom("pom.xml").toString();
                println("versionInfo without snapshot extension :" +version);
            }

            echo "version:  ${version}"

            // Set the tag for the production image: version -- to be discussed
            def prodTag = "${version}"
            echo "prodTag: ${prodTag}"

            // Set the tag for the development image: version + build number
            def devTag  = "${version}-${commitId}"
            echo "devTag: ${devTag}"

            def path = getGroupIdFromPom("pom.xml").toString().replace(".", "/")
            echo "package path: ${path}"

            // Using Maven build the war file
            // Do not run tests in this step
            stage('Build application binary') {
                echo "Building version ${devTag}"
                sh "${mvnCmd} clean package -DskipTests=true"

            }

            // Using Maven run the unit tests
            stage('Unit Tests') {
                echo "Running Unit Tests"
                sh "${mvnCmd} test -DskipTests=true"
            }

            // Using Maven call SonarQube for Code Analysis
            stage('Code Analysis') {
                echo "Running Code Analysis"
                sh "${mvnCmd} sonar:sonar -Dsonar.projectName=${SERVICE_NAME} -Dsonar.projectVersion=${devTag}"
            }

            // Publish the built war file to Nexus
            stage('Publish to Nexus') {
                echo "Publish to Nexus"

                sh "${mvnCmd} deploy:deploy-file -DgroupId=${groupId} -DartifactId=${artifactId} -Dversion=${devTag} -Dpackaging=jar -DrepositoryId=nexus -Durl=http://nexus3.cicd.svc.cluster.local:8081/repository/releases -Dfile=target/${artifactId}-${pomVersion}.jar -DskipTests=true -DpomFile=pom.xml"
            }

            // Create or replace Image build artifacts
            stage('Create Image Builder') {
                openshift.withCluster(){
                    openshift.withProject( "cicd" ){
                
                echo "Creating BuildConfig"
                sh "if oc get bc ${SERVICE_NAME} --namespace=cicd; \
                    then echo \"exist\"; \
                    else oc new-build --binary=true --name=${SERVICE_NAME} ${BUILD_IMAGE} --labels=app=${SERVICE_NAME} -n cicd;fi"                            
            }    
              }
          }


            // Build the OpenShift Image in OpenShift and tag it.
            stage('Build and Tag OpenShift Image') {
                openshift.withCluster(){
                    openshift.withProject( "cicd" ){
                echo "Building OpenShift container image ${SERVICE_NAME}:${devTag}"

                // Start Binary Build in OpenShift CICD cluster using the file we just published
                sh "oc start-build ${SERVICE_NAME} --follow --from-file=target/${artifactId}-${pomVersion}.jar -n cicd" 
                echo "oc start build complete."
                
                // use the file you just published into Nexus
                // sh "oc start-build ${SERVICE_NAME} --follow --from-file=http://nexus3.cicd.svc.cluster.local:8081/repository/releases/${path}/${artifactId}/${version}/${artifactId}-${version}.jar -n cicd"
				// Tag the image using the devTag
                openshiftTag alias: 'false', destStream: SERVICE_NAME, destTag: devTag, destinationNamespace: 'cicd', namespace: 'cicd', srcStream: SERVICE_NAME, srcTag: 'latest', verbose: 'false'

            }
             }

            }
			

            // // Deploy the built image to the Development Environment.
            stage('Configure Deployment') {
                echo "Deploying container image to Development Project"
				openshift.withCluster(){
					echo "Using openshift cluster ${openshift.cluster()}"
                    
					openshift.withProject( "${DEV_PROJECT}" ) {
        					echo "Using project: ${openshift.project()}"
							//Create application if it does not exist
							def deploymentSelector = openshift.selector( "dc", "${SERVICE_NAME}")
    						def deploymentExists = deploymentSelector.exists()
							if (!deploymentExists) {
								echo "Deployment ${SERVICE_NAME} does not exist"
								openshift.newApp("${DEV_PROJECT}/${SERVICE_NAME}:0.0.0", "--name=${SERVICE_NAME}", "--allow-missing-imagestream-tags=true","--namespace=${DEV_PROJECT}")
								
							}
							echo "Deployment ${SERVICE_NAME} exists"

							// Disable triggers, we want to control manually
							openshift.raw("set","triggers","dc/${SERVICE_NAME}","--remove-all","--namespace","${DEV_PROJECT}")

							// Set resource requirements
							try {
								def result = openshift.raw("set","resources","dc/${SERVICE_NAME}","--limits=cpu=${CPU_LIMITS},memory=${MEM_LIMITS}","--requests=cpu=${CPU_REQUESTS},memory=${MEM_REQUESTS}","--namespace=${DEV_PROJECT}")
							} catch(e) {
								echo e.getMessage()
								if(!e.getMessage().contains("info:")){
									throw e
								}
							}

							
							//Expose service
							try {
								openshift.raw("expose","dc ${SERVICE_NAME}","--port 8080","-n ${DEV_PROJECT}")
							} catch(e) {
								echo e.getMessage()
								if(!e.getMessage().contains("AlreadyExists")){
									throw e
								}
							}

							//Update the Image on the Development Deployment Config
							openshift.raw("set","image dc/${SERVICE_NAME}","${SERVICE_NAME}=docker-registry.default.svc:5000/${DEV_PROJECT}/${SERVICE_NAME}:${devTag}","-n ${DEV_PROJECT}")
							
    				}

				}

			}


			
			stage("Rollout to Developmemt"){
				openshift.withCluster(){
					openshift.withProject( "${DEV_PROJECT}" ) {
						def deployment = openshift.selector('dc', "${SERVICE_NAME}")
						deployment.rollout().latest()
						//rollout.status()
					}

				}


			}


        }

    }
}

  // Convenience Functions to read variables from the pom.xml
  // Do not change anything below this line.
  def getVersionFromPom(pom) {
      def matcher = readFile(pom) =~ '<version>(.+)</version>'
      matcher ? matcher[0][1] : null
  }
  def getGroupIdFromPom(pom) {
      def matcher = readFile(pom) =~ '<groupId>(.+)</groupId>'
      matcher ? matcher[0][1] : null
  }
  def getArtifactIdFromPom(pom) {
      def matcher = readFile(pom) =~ '<artifactId>(.+)</artifactId>'
      matcher ? matcher[0][1] : null
  }      
 