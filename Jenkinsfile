pipeline {
	agent any

	stages {

	     stage ('Compile') {
		steps {
		  withMaven(maven : 'maven') {
			sh 'mvn clean compile'

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
		
		
stage("Build & SonarQube analysis") {
          steps {
              withSonarQubeEnv('Sonar') {
                 sh 'mvn clean package sonar:sonar'
              }
          }
      }		
		
	stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
	
		}
	


}
