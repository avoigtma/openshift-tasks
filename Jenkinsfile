#!groovy

node {
  echo "Global Jenkins node"
  
//  sh 'env > env1.txt'
//  String[] envs = readFile('env1.txt').split("\r?\n")

//  for(String vars: envs){
//    println(vars)
//  }
}



node('maven') {
  echo "Jenkins Maven node"
  
//  sh 'env > env2.txt'
//  String[] envs = readFile('env2.txt').split("\r?\n")

//  for(String vars: envs){
//    println(vars)
//  }
}

// Run this node on a Maven Slave
// Maven Slaves have JDK and Maven already installed
node('maven') {
  // Make sure your nexus_openshift_settings.xml
  // Is pointing to your nexus instance
  //def mvnCmd = "mvn -s ./nexus_openshift_settings.xml"
  def mvnCmd = "mvn -DMAVEN_MIRROR_URL=http://moe:8081/content/groups/public"

  stage('Checkout Source') {
    // Get Source Code from SCM (Git) as configured in the Jenkins Project
    // Next line for inline script, "checkout scm" for Jenkinsfile from Gogs
    //git 'http://gogs.xyz-gogs.svc.cluster.local:3000/CICDLabs/openshift-myproject.git'
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
    //...
  }
  stage('Code Analysis') {
    echo "Code Analysis"
    //...
  }
  stage('Publish to Nexus') {
    echo "Publish to Nexus"
    //...
  }

  stage('Build OpenShift Image') {
      def newTag = "DevCandidate-${version}"
      echo "New Tag: ${newTag}"

      // Replace myproject-dev with the name of your dev project
      openshiftBuild bldCfg: 'tasks', checkForTriggeredDeployments: 'false', namespace: 'myproject-dev', showBuildLogs: 'false', verbose: 'false', waitTime: '', waitUnit: 'sec'
      openshiftVerifyBuild bldCfg: 'tasks', checkForTriggeredDeployments: 'false', namespace: 'myproject-dev', verbose: 'false', waitTime: ''
      openshiftTag alias: 'false', destStream: 'tasks', destTag: newTag, destinationNamespace: 'myproject-dev', namespace: 'myproject-dev', srcStream: 'tasks', srcTag: 'latest', verbose: 'false'
  }

  stage('Deploy to Dev') {
    // Patch the DeploymentConfig so that it points to the latest TestingCandidate-${version} Image.
    // Replace myproject-dev with the name of your dev project
    sh "oc project myproject-dev"
    sh "oc patch dc tasks --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"tasks\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"myproject-dev\", \"name\": \"tasks:TestingCandidate-$version\"}}}]}}' -n myproject-dev"

      openshiftDeploy depCfg: 'tasks', namespace: 'myproject-dev', verbose: 'false', waitTime: '', waitUnit: 'sec'
      openshiftVerifyDeployment depCfg: 'tasks', namespace: 'myproject-dev', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
      openshiftVerifyService namespace: 'myproject-dev', svcName: 'tasks', verbose: 'false'
  }

  stage('Deploy to Test') {
      def newTag = "TestingCandidate-${version}"
      echo "New Tag: ${newTag}"
    
    requestUserInput('Release to TEST?')

    // Patch the DeploymentConfig so that it points to the latest TestingCandidate-${version} Image.
    // Replace myproject-dev with the name of your dev project
    sh "oc project myproject-test"
    sh "oc patch dc tasks --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"tasks\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"myproject-dev\", \"name\": \"tasks:TestingCandidate-$version\"}}}]}}' -n myproject-test"

      openshiftDeploy depCfg: 'tasks', namespace: 'myproject-test', verbose: 'false', waitTime: '', waitUnit: 'sec'
      openshiftVerifyDeployment depCfg: 'tasks', namespace: 'myproject-test', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
      openshiftVerifyService namespace: 'myproject-test', svcName: 'tasks', verbose: 'false'
  }

  stage('Integration Test') {
    // ...

    def newTag = "ProdReady-${version}"
    echo "New Tag: ${newTag}"

    // Replace myproject-dev with the name of your dev project
    openshiftTag alias: 'false', destStream: 'tasks', destTag: newTag, destinationNamespace: 'myproject-dev', namespace: 'myproject-dev', srcStream: 'tasks', srcTag: 'latest', verbose: 'false'
  }

  // Blue/Green Deployment into Production
  // -------------------------------------
  def dest   = "tasks-green"
  def active = ""

  stage('Prep Production Deployment') {
    // Replace myproject-dev and myproject-test with
    // your project names
    sh "oc project myproject-test"
    sh "oc get route tasks -n myproject-test -o jsonpath='{ .spec.to.name }' > activesvc.txt"
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
    // Replace myproject-dev and myproject-test with
    // your project names.
    sh "oc patch dc ${dest} --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"$dest\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"myproject-dev\", \"name\": \"myproject:ProdReady-$version\"}}}]}}' -n myproject-test"

    openshiftDeploy depCfg: dest, namespace: 'myproject-test', verbose: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyDeployment depCfg: dest, namespace: 'myproject-test', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'true', waitTime: '', waitUnit: 'sec'
    openshiftVerifyService namespace: 'myproject-test', svcName: dest, verbose: 'false'
  }
  stage('Switch over to new Version') {
    input "Switch Production?"

    // Replace myproject-test with the name of your
    // production project
    sh 'oc patch route tasks -n myproject-test -p \'{"spec":{"to":{"name":"' + dest + '"}}}\''
    sh 'oc get route tasks -n myproject-test > oc_out.txt'
    oc_out = readFile('oc_out.txt')
    echo "Current route configuration: " + oc_out
  }
}

def requestUserInput(message) {
  try {
      timeout(time: 15, unit: 'SECONDS') {
          input message: 'Do you want to release this build?',
                parameters: [[$class: 'BooleanParameterDefinition',
                              defaultValue: false,
                              description: 'Ticking this box will do a release',
                              name: 'Release']]
      }
  } catch (err) {
      def user = err.getCauses()[0].getUser()
      echo "Aborted by:\n ${user}"

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
