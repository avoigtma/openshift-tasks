#!groovy

// Run this node on a Maven Slave
// Maven Slaves have JDK and Maven already installed
node('maven') {
  // Make sure your nexus_openshift_settings.xml
  // Is pointing to your nexus instance
  def mvnCmd = "mvn -s ./nexus_openshift_settings.xml"

  stage('Checkout Source') {
    // Get Source Code from SCM (Git) as configured in the Jenkins Project
    // Next line for inline script, "checkout scm" for Jenkinsfile from Gogs
    //git 'http://gogs.xyz-gogs.svc.cluster.local:3000/CICDLabs/openshift-tasks.git'
    checkout scm
  }

  // The following variables need to be defined at the top level and not inside
  // the scope of a stage - otherwise they would not be accessible from other stages.
  // Extract version and other properties from the pom.xml
  def groupId    = getGroupIdFromPom("pom.xml")
  def artifactId = getArtifactIdFromPom("pom.xml")
  def version    = getVersionFromPom("pom.xml")

  stage('Build war') {
    echo "Building version ${version}"

    sh "${mvnCmd} clean package -DskipTests"
  }
  stage('Unit Tests') {
    echo "Unit Tests"
    sh "${mvnCmd} test"
  }
  stage('Code Analysis') {
    echo "Code Analysis"

    // Replace xyz-sonarqube with the name of your project
    sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube.xyz-sonarqube.svc.cluster.local:9000/ -Dsonar.projectName=${JOB_BASE_NAME}"
  }
  stage('Publish to Nexus') {
    echo "Publish to Nexus"

    // Replace xyz-nexus with the name of your project
    sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3.xyz-nexus.svc.cluster.local:8081/repository/releases"
    // Get Repository from Git/Gogs
    // Replace xyz-gogs with the name of your Gogs project
    git url: 'http://<gogs_user>:<gogs_password>@gogs.xyz-gogs.svc.cluster.local:3000/CICDLabs/openshift-tasks-ocp.git'

    // Create war file name from pom.xml properties
    String warFileName = "${groupId}.${artifactId}"
    warFileName = warFileName.replace('.', '/')
    sh "echo ${warFileName}/${version}/${artifactId}-${version}.war"

    // Update the .s2i/environment file with the location of the latest war file name
    // Also add Build Number so that every build we get a unique environment file and
    // The git commit/push step succeeds.
    // Replace xyz-nexus with the name of your project
    sh "echo WAR_FILE_LOCATION=http://nexus3.xyz-nexus.svc.cluster.local:8081/repository/releases/${warFileName}/${version}/${artifactId}-${version}.war >.s2i/environment"
    sh "echo BUILD_NUMBER=${BUILD_NUMBER} >>.s2i/environment"

    // Update the Git/Gogs repository with the latest file
    // Replace XYZ with your Initials
    def commit = "Release " + version
    sh "git config --global user.email xyz@example.opentlc.com && git config --global user.name XYZJenkins"
    sh "git add .s2i/environment && git commit -m \"${commit}\" && git push origin master"
  }

    stage('Build OpenShift Image') {
      def newTag = "TestingCandidate-${version}"
      echo "New Tag: ${newTag}"

      // Replace tasks-dev with the name of your dev project
      openshiftBuild bldCfg: 'tasks', checkForTriggeredDeployments: 'false', namespace: 'tasks-dev', showBuildLogs: 'false', verbose: 'false', waitTime: '', waitUnit: 'sec'
      openshiftVerifyBuild bldCfg: 'tasks', checkForTriggeredDeployments: 'false', namespace: 'tasks-dev', verbose: 'false', waitTime: ''
      openshiftTag alias: 'false', destStream: 'tasks', destTag: newTag, destinationNamespace: 'tasks-dev', namespace: 'tasks-dev', srcStream: 'tasks', srcTag: 'latest', verbose: 'false'
  }

  stage('Deploy to Dev') {
    // Patch the DeploymentConfig so that it points to the latest TestingCandidate-${version} Image.
    // Replace tasks-dev with the name of your dev project
    sh "oc project tasks-dev"
    sh "oc patch dc tasks --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"tasks\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"tasks-dev\", \"name\": \"tasks:TestingCandidate-$version\"}}}]}}' -n tasks-dev"

      openshiftDeploy depCfg: 'tasks', namespace: 'tasks-dev', verbose: 'false', waitTime: '', waitUnit: 'sec'
      openshiftVerifyDeployment depCfg: 'tasks', namespace: 'tasks-dev', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
      openshiftVerifyService namespace: 'tasks-dev', svcName: 'tasks', verbose: 'false'
  }

  stage('Integration Test') {
    // TBD: Proper test
    // Could use the OpenShift-Tasks REST APIs to make sure it is working as expected.

    def newTag = "ProdReady-${version}"
    echo "New Tag: ${newTag}"

    // Replace tasks-dev with the name of your dev project
    openshiftTag alias: 'false', destStream: 'tasks', destTag: newTag, destinationNamespace: 'tasks-dev', namespace: 'tasks-dev', srcStream: 'tasks', srcTag: 'latest', verbose: 'false'
  }

  // Blue/Green Deployment into Production
  // -------------------------------------
  def dest   = "tasks-green"
  def active = ""

  stage('Prep Production Deployment') {
    // Replace tasks-dev and tasks-prod with
    // your project names
    sh "oc project tasks-prod"
    sh "oc get route tasks -n tasks-prod -o jsonpath='{ .spec.to.name }' > activesvc.txt"
    active = readFile('activesvc.txt').trim()
    if (active == "tasks-green") {
      dest = "tasks-blue"
    }
    echo "Active svc: " + active
    echo "Dest svc:   " + dest
  }
  stage('Deploy new Version') {
    echo "Deploying to ${dest}"

    // Patch the DeploymentConfig so that it points to
    // the latest ProdReady-${version} Image.
    // Replace tasks-dev and tasks-prod with
    // your project names.
    sh "oc patch dc ${dest} --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"$dest\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"tasks-dev\", \"name\": \"tasks:ProdReady-$version\"}}}]}}' -n tasks-prod"

    openshiftDeploy depCfg: dest, namespace: 'tasks-prod', verbose: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyDeployment depCfg: dest, namespace: 'tasks-prod', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'true', waitTime: '', waitUnit: 'sec'
    openshiftVerifyService namespace: 'tasks-prod', svcName: dest, verbose: 'false'
  }
  stage('Switch over to new Version') {
    input "Switch Production?"

    // Replace tasks-prod with the name of your
    // production project
    sh 'oc patch route tasks -n tasks-prod -p \'{"spec":{"to":{"name":"' + dest + '"}}}\''
    sh 'oc get route tasks -n tasks-prod > oc_out.txt'
    oc_out = readFile('oc_out.txt')
    echo "Current route configuration: " + oc_out
  }
}

// Convenience Functions to read variables from the pom.xml
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
