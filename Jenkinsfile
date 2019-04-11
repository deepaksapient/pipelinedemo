pipeline {
    agent any

    parameters {
         string(name: 'tomcat_dev', defaultValue: 'http://10.0.1.13:9090/', description: 'Dev Server')
    }

    triggers {
         pollSCM('* * * * *')
     }

stages{
    stage('Build'){
            steps {
                sh 'mvn clean package'
            }            
        }
	stage ('Executing Test Cases') {
		steps {
		  withMaven(maven : 'maven') {
			sh 'mvn test'

			}
		  }

		}	
              
                stage ('Deploy to Dev'){
                    steps {
                        sh "sudo -u tomcat cp /var/lib/jenkins/workspace/NewPipelineDemo/target/*.war /var/lib/tomcat/webapps/"
			//sh "scp **/target/*.war /var/lib/tomcat/webapps/"
                    }
                }  
	
	stage("SonarQube analysis") {
          steps {
              withSonarQubeEnv('Sonar') {
                 sh 'mvn sonar:sonar -Dsonar.host.url=http://10.0.1.13:9000 -Dsonar.login=2546598c8d5b9277e3d92140ab70b207c2002e7e'
              }
          }
      }		
        
	stage ('Artifactory configuration') {
            steps {
                rtUpload (
    serverId: "Pipelinedemo",
    spec:
        """{
          "files": [
            {
              "pattern": "*.war",
              "target": "pipelinedemo"
            }
         ]
        }"""
)
            }
        }
	
    }
}
