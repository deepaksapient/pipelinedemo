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
			sh 'mvn packages'

			}
		  }

		}
	}


}
