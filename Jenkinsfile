properties([
    parameters([
	choice(name: "TYPE", choices: ["java-11", "java-17", "java-18"], description: "LANGUAGES"),
        choice(name: "SERVICE", choices: ["gembook"], description: "services to be build"),
        choice(name: "PORT", choices: ["8081", "80", "8080", "8999", "7000"], description: "port to be used"),
    ])
])

// env.BRANCH="*/main"
env.MAINTAINER= "maintainer@geminisolutions.com"
env.BRANCH_CENTRALIZED_FILES="*/main"
env.SSH_LINK_CENTRALIZED_FILES="git@github.com:gem-siddhant/centralized-repo.git"

if(params.SERVICE=="gembook")
{
    env.SSH_LINK= 'git@github.com:Gemini-Solutions/GembookSvc.git'
    env.BRANCH= "*/centralized-files-test*"
}

env.REGISTRY= params.SERVICE.toLowerCase()
env.TRIVY_NODE = 'image_builder_trivy'
env.TRIVY_CONTAINER = 'docker-image-builder-trivy'

if(params.TYPE == "java-11")
{
    env.NODE_NAME = 'maven_runner_java11'
    env.CONTAINER_NAME = 'maven-runner-11'
    env.STAGE_NAME = 'maven_Build'
    env.CMD1= 'rm -rf target'
    env.CMD2= 'mvn package'
    env.IMAGE='adoptopenjdk\\/openjdk11'
    env.WORKDIR_CMD= '\\/home\\/'
}

node("${env.NODE_NAME}") {
      stage('Repo_Checkout') {
             dir ('repo') {
             checkout([$class: 'GitSCM', branches: [[name: "$BRANCH"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg:  [], \
    userRemoteConfigs: [[credentialsId: 'admingithub', url: "$SSH_LINK", poll: 'false']]])
             }
      }
	stage("${env.STAGE_NAME}") {
	      container("${env.CONTAINER_NAME}") {
            dir ('repo'){
		    if (env.NODE_NAME == 'maven_runner_java11'  || env.NODE_NAME == 'maven_runner_java17' || env.NODE_NAME == 'maven-runner-18')
		    {
                    	sh 'env'
		    	sh "${env.CMD1}"
 		    	sh "${env.CMD2}"
			dir ('target'){
		    		sh 'pwd' 
 		    		sh 'chmod +x *.?ar'
// 		    		sh 'env.EXECUTOR=`ls *.[j,w]ar`'
// 		    		sh "sed -i -e 's/EXECUTOR/$EXECUTOR/g' ../DockerFile"
			}
            	}
           }
        }
    }
}


//Docker and Deployment Stage
node("${env.TRIVY_NODE}") {
       try {
	stage('Checkout_Centralized_Files') {
           dir ('repo') {
               checkout([$class: 'GitSCM', 
               branches: [[name: "$BRANCH_CENTRALIZED_FILES"]], 
               doGenerateSubmoduleConfigurations: false, 
               extensions: [], 
               submoduleCfg:  [], 
               userRemoteConfigs: [[credentialsId: 'admingithub', 
               url: "$SSH_LINK_CENTRALIZED_FILES", 
               poll: 'false']]])
           }
       }
       stage('Build_image') {
                dir ('repo') {
			container("${env.TRIVY_CONTAINER}") {
                  withCredentials([usernamePassword(credentialsId: 'docker_registry', passwordVariable: 'docker_pass', usernameVariable: 'docker_user')]) {
                //   sh 'echo TYPE is : $SERVICE'
		        //   sh 'sed -i -e "s/SERVICE/$SERVICE/g" Dockerfile deployment-type.yaml' 
                  sh 'sed -i -e "s/SERVICE/$SERVICE/g" -e "s/PORT/$PORT/g"  -e "s/REGISTRY/$REGISTRY/g" -e "s/IMAGE/$IMAGE/g" -e "s/WORKDIR_CMD/$WORKDIR_CMD/g" DockerFile Deployment-beta.yaml' 
		  sh 'cat DockerFile'	  
                  sh 'docker image build -f DockerFile --build-arg REGISTRY=$REGISTRY -t registry-np.geminisolutions.com/$REGISTRY/$REGISTRY:1.0-$BUILD_NUMBER -t registry-np.geminisolutions.com/$REGISTRY/$REGISTRY .'
                  sh 'trivy image -f json registry-np.geminisolutions.com/$REGISTRY/$REGISTRY:1.0-$BUILD_NUMBER > trivy-report.json' 
	           archiveArtifacts artifacts: 'trivy-report.json', onlyIfSuccessful: true
                  sh '''docker login -u $docker_user -p $docker_pass https://registry-np.geminisolutions.com'''
                  sh 'docker push registry-np.geminisolutions.com/$REGISTRY/$REGISTRY:1.0-$BUILD_NUMBER'
                  sh 'docker push registry-np.geminisolutions.com/$REGISTRY/$REGISTRY'
                  sh 'rm -rf build/'
               }
             }
          }
       }
       stage('Deployment_stage') {
               dir ('repo') {
                   container("${env.TRIVY_CONTAINER}") {
                   kubeconfig(credentialsId: 'KubeConfigCred') {
                   sh '/usr/local/bin/kubectl apply -f Deployment-beta.yaml -n dev'
                   sh '/usr/local/bin/kubectl rollout restart Deployment $REGISTRY -n dev'

                   }
                }
            }
        }
    } finally {
         //sh 'echo current_image="registry-np.geminisolutions.com/helpdesk/server:1.0-$BUILD_NUMBER" > build.properties'
         //archiveArtifacts artifacts: 'build.properties', onlyIfSuccessful: true
    }
}
