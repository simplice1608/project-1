currentBuild.displayName = "Build # "+currentBuild.number

   def getDockerTag(){
        def tag = sh script: 'git rev-parse HEAD', returnStdout: true
        return tag
        }

pipeline{

   agent { label 'node1' }
   environment{
	    Docker_tag = getDockerTag()
        }
   tools{

      maven '3.9.0'
   }

   stages{
       stage("Checkout"){
        steps{
             git url: 'https://github.com/simplice1608/project-1.git', branch: 'main'
             sh "ls -ll"
        }
       }
       stage("UnitTest"){
	    agent {
          docker { image 'maven:3.8.1-adoptopenjdk-11' }
        }
        steps{
            sh "mvn test"
        }
       }
       stage('Quality Gate Statuc Check'){

               agent {
                docker {
                image 'maven'
                args '-v $HOME/.m2:/root/.m2'
                }
            }
                  steps{
                      script{
                      withSonarQubeEnv('sonar') { 
                      sh "mvn sonar:sonar"
                       }
                      timeout(time: 1, unit: 'HOURS') {
                      def qg = waitForQualityGate()
                      if (qg.status != 'OK') {
                           error "Pipeline aborted due to quality gate failure: ${qg.status}"
                      }
                    }
		    sh "mvn clean install"
                  }
                }  
              }
       stage("Build"){
// 	agent {
//         docker { image 'maven:3.8.1-adoptopenjdk-11' }
//          }

        steps{

            sh 'mvn package'
        }
       }
       stage("ArchiveArtifacts"){

        steps{
               archiveArtifacts artifacts: 'target/WebApp.war', followSymlinks: false
        }
       }
       stage("NexusPublisher"){

        steps{

            echo "Uploading artifacts to Nexus"
        }
       }

       stage("Deploy Dev Env"){

        steps{

            sshPublisher(publishers: [sshPublisherDesc(configName: 'dev-tomcat', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: '', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: 'target', sourceFiles: 'target/*.war')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
        }
       }

       stage("Approval"){

        steps{

            timeout(time: 15, unit: 'MINUTES'){ 
	               input message: 'Do you approve deployment for production?' , ok: 'Yes'}

        }
       }

       stage("Prod build"){
        steps{
            script{
                   sh 'docker build . -t simplice1608/docker-project1:$Docker_tag'
		          withCredentials([string(credentialsId: 'docker', variable: 'docker_password')]) {		    
				  sh 'docker login -u simplice1608 -p $docker_password'
				  sh 'docker push simplice1608/docker-project1:$Docker_tag'
			}
                       }
                    }
        }

       stage('ansible playbook'){
	                agent { label 'node1' }
			steps{
			 	script{
				    sh '''final_tag=$(echo $Docker_tag | tr -d ' ')
				     echo ${final_tag}test
				     sed -i "s/docker_tag/$final_tag/g"  deployment.yaml
				     '''
				    ansiblePlaybook become: true, installation: 'ansible', inventory: 'hosts', playbook: 'ansible.yaml'
				}
			}
		}
       }

   post{
        always {
            slackSend( color: "good", message: "I have done my job")
        }
        success{
            slackSend( color: "good", message: "${custom_msg()}")
        }
        failure{
            slackSend( color: "danger", message: "${custom_msg()}")
        }


}

}
def custom_msg()
{
  def JENKINS_URL= "http://3.92.179.42:8080/"
  def JOB_NAME = env.JOB_NAME
  def BUILD_ID= env.BUILD_ID
  def JENKINS_LOG= " Success: Job [${env.JOB_NAME}] Logs path: ${JENKINS_URL}/job/${JOB_NAME}/${BUILD_ID}/consoleText"
  return JENKINS_LOG
}
