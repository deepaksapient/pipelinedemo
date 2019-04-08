#!groovy
import groovy.json.JsonOutput
import groovy.json.JsonSlurper

test_env=http://10.0.1.13:8080
prod_env=http://10.0.1.13:8080

node {

     server = Artifactory.server "artifactory"
     buildInfo = Artifactory.newBuildInfo()
     buildInfo.env.capture = true
    
    // we need to set a newer JVM for Sonar
    env.JAVA_HOME="${tool 'Java SE DK 8u131'}"
    env.PATH="${env.JAVA_HOME}/bin:${env.PATH}"
    
    // pull request or feature branch
    if  (env.BRANCH_NAME != 'master') {
        checkout()
        build()
        unitTest()
        testDeployment()
        sonarServer()
		sonarPreview()
        allCodeQualityTests()      
    } 
	// master branch / production
    else { 
        checkout()
        build()
        allTests()
        testDeployment()
        sonarServer()
		sonarPreview()
        allCodeQualityTests()
        preProduction()
        production()
    }
}


def checkout () {
    stage 'Checkout code'
    context="continuous-integration/jenkins/"    
    checkout scm
    setBuildStatus ("${context}", 'Checking out completed', 'SUCCESS')
}

def build () {
    stage 'Build'
    mvn 'clean install -DskipTests=true -Dmaven.javadoc.skip=true -Dcheckstyle.skip=true -B -V'
}


def unitTest() {
    stage 'Unit tests'
    mvn 'test -B -Dmaven.javadoc.skip=true -Dcheckstyle.skip=true'
    if (currentBuild.result == "UNSTABLE") {
        sh "exit 1"
    }
}

def allTests() {
    stage 'All tests'
    // don't skip anything
    mvn 'test -B'
    step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
    if (currentBuild.result == "UNSTABLE") {
        sh "exit 1"
    }
}

def testDeployment() {
    stage name: 'Deploy to Test env', concurrency: 1
    def testapp = "${env.test_env}"
    def id = createDeployment(getBranch(), "preview", "Deploying branch to test")
    echo "Deployment ID: ${id}"
    if (id != null) {
        setDeploymentStatus(id, "pending", "https://${testapp}.testapp.com/", "Pending deployment to test");
        testDeploy "${testapp}"
        setDeploymentStatus(id, "success", "https://${testapp}.testapp.com/", "Successfully deployed to test");
    }
    mvn 'deploy -DskipTests=true'
}

def sonarPreview() {
    stage('SonarQube Preview') {
        prNo = (env.BRANCH_NAME=~/^PR-(\d+)$/)[0][1]
        mvn "org.jacoco:jacoco-maven-plugin:prepare-agent install -Dmaven.test.failure.ignore=true -Pcoverage-per-test"
        withCredentials([[$class: 'StringBinding', credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN']]) {
            githubToken=env.GITHUB_TOKEN
            repoSlug=getRepoSlug()
            withSonarQubeEnv('SonarQube Octodemoapps') {
                mvn "-Dsonar.analysis.mode=preview -Dsonar.github.pullRequest=${prNo} -Dsonar.github.oauth=${githubToken} -Dsonar.github.repository=${repoSlug} -Dsonar.github.endpoint=https://api.github.com/ org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar"
            }
        }
    } 
}
    
def sonarServer() {
    stage('SonarQube Server') {
        mvn "org.jacoco:jacoco-maven-plugin:prepare-agent install -Dmaven.test.failure.ignore=true -Pcoverage-per-test"
        withSonarQubeEnv('SonarQube Octodemoapps') {
            mvn "org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar"
        }
        
        context="sonarqube/qualitygate"
        setBuildStatus ("${context}", 'Checking Sonarqube quality gate', 'PENDING')
        timeout(time: 1, unit: 'MINUTES') { // Just in case something goes wrong, pipeline will be killed after a timeout
            def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
            if (qg.status != 'OK') {
                setBuildStatus ("${context}", "Sonarqube quality gate fail: ${qg.status}", 'FAILURE')
                error "Pipeline aborted due to quality gate failure: ${qg.status}"
            } else {
                setBuildStatus ("${context}", "Sonarqube quality gate pass: ${qg.status}", 'SUCCESS')
            }    
        }
    }
}

def allCodeQualityTests() {
    stage 'Code Quality'
    lintTest()
    coverageTest()
}

def lintTest() {
    context="continuous-integration/jenkins/linting"
    setBuildStatus ("${context}", 'Checking code conventions', 'PENDING')
    lintTestPass = true

    try {
        mvn 'verify -DskipTests=true'
    } catch (err) {
        setBuildStatus ("${context}", 'Some code conventions are broken', 'FAILURE')
        lintTestPass = false
    } finally {
        if (lintTestPass) setBuildStatus ("${context}", 'Code conventions OK', 'SUCCESS')
    }
}

def coverageTest() {
    context="continuous-integration/jenkins/coverage"
    setBuildStatus ("${context}", 'Checking code coverage levels', 'PENDING')

    coverageTestStatus = true

    try {
        mvn 'cobertura:check'
    } catch (err) {
        setBuildStatus("${context}", 'Code coverage below 90%', 'FAILURE')
        throw err
    }

    setBuildStatus ("${context}", 'Code coverage above 90%', 'SUCCESS')

}


def preProduction() {
    stage name: 'Deploy to Pre-Production', concurrency: 1
    switchSnapshotBuildToRelease()
    buildAndPublishToArtifactory()
}

def switchSnapshotBuildToRelease() {
    def descriptor = Artifactory.mavenDescriptor()
    descriptor.version = '1.0.0'
    descriptor.pomFile = 'pom.xml'
    descriptor.transform()
}

def buildAndPublishToArtifactory() {       
        def rtMaven = Artifactory.newMavenBuild()
        rtMaven.tool = "Maven 3.x"
        rtMaven.deployer releaseRepo:'libs-release-local', snapshotRepo:'libs-snapshot-local', server: server
        rtMaven.resolver releaseRepo:'libs-release', snapshotRepo:'libs-snapshot', server: server
        rtMaven.run pom: 'pom.xml', goals: 'install', buildInfo: buildInfo
        server.publishBuildInfo buildInfo
}
def production() {
    stage name: 'Deploy to Production', concurrency: 1
    step([$class: 'ArtifactArchiver', artifacts: '**/target/*.jar', fingerprint: true])
    testDeploy "${env.prod_env}"
    def version = getCurrentReleaseVersion("${env.prod_env}")
    def createdAt = getCurrentReleaseDate("${env.prod_env}", version)
    echo "Release version: ${version}"
    createRelease(version, createdAt)
    promoteBuildInArtifactory()
	distributeBuildToBinTray()
}

def promoteBuildInArtifactory() {
        def promotionConfig = [
            // Mandatory parameters
            'buildName'          : buildInfo.name,
            'buildNumber'        : buildInfo.number,
            'targetRepo'         : 'libs-prod-local',
 
            // Optional parameters
            'comment'            : 'deploying to production',
            'sourceRepo'         : 'libs-release-local',
            'status'             : 'Released',
            'includeDependencies': false,
            'copy'               : true,
            // 'failFast' is true by default.
            // Set it to false, if you don't want the promotion to abort upon receiving the first error.
            'failFast'           : true
        ]
 
        // Promote build
        server.promote promotionConfig
}

def distributeBuildToBinTray() {
        def distributionConfig = [
            'buildName'             : buildInfo.name,
            'buildNumber'           : buildInfo.number,
            'targetRepo'            : 'reading-time-dist',  
            // Optional parameters
            //'publish'               : true, // Default: true. If true, artifacts are published when deployed to Bintray.
            'overrideExistingFiles' : true, // Default: false. If true, Artifactory overwrites builds already existing in the target path in Bintray.
            //'gpgPassphrase'         : 'passphrase', // If specified, Artifactory will GPG sign the build deployed to Bintray and apply the specified passphrase.
            //'async'                 : false, // Default: false. If true, the build will be distributed asynchronously. Errors and warnings may be viewed in the Artifactory log.
            //"sourceRepos"           : ["yum-local"], // An array of local repositories from which build artifacts should be collected.
            //'dryRun'                : false, // Default: false. If true, distribution is only simulated. No files are actually moved.
        ]
        server.distribute distributionConfig
}

