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

stage ('Deployment') {
		steps {
		  withMaven(maven : 'maven') {
			sh 'mvn package'

			}
		  }
}
	stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    // true = set pipeline to UNSTABLE, false = don't
                    // Requires SonarQube Scanner for Jenkins 2.7+
                    waitForQualityGate abortPipeline: true
                }
            }
        }
	
		}
	


}
