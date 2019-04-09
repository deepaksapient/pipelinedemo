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
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
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
                        sh "cp **/target/*.war /var/lib/tomcat/webapps/"
			//sh "scp **/target/*.war /var/lib/tomcat/webapps/"
                    }
                }            
        
    }
}
