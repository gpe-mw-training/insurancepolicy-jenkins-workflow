stage 'Build'
node {
    echo "Groovy:  Build stage:  ";
    checkPort("gitlab", "22");
    git url: 'ssh://git@gitlab:22/acme-insurance/insurancepolicy.git', credentialsId: 'jenkins'
    def version = getBuildVersion("policyquote/pom.xml")
    env.BUILD_VERSION = version
    env.BUILD_GROUP_ID = getGroupIdFromPom("policyquote/pom.xml")
    env.BUILD_ARTIFACT_ID = getArtifactIdFromPom("policyquote/pom.xml")
    def branch = 'build-' + version
    env.BUILD_BRANCH = branch
    prepareBuild(version, branch)
    build()
    stash excludes: 'target/', includes: '**', name: 'source'
}

stage 'Integrate'
node {
    unstash 'source'
    integrationTests()    
}

stage 'Publish'
node {
    unstash 'source'
    publishToNexusAndCommitBranch(env.BUILD_VERSION, env.BUILD_BRANCH)
}

stage 'QA'
node {
    checkPort("bpmsqa", "8080");
    deployToBPMS("bpmsqa:8080")
}

stage 'Approve'
timeout(time: 2, unit: 'DAYS') {
    input message: 'Do you want to deploy into production?'
}

stage 'Production'
node {
    checkPort("bpmsprod", "8080");
    deployToBPMS("bpmsprod:8080")
}

def checkPort(server, port) {
  sh "checkPort ${server} ${port}"
}

def prepareBuild(version, branch) {
    sh "git checkout -b ${branch}"
    withEnv(["PATH+MAVEN=${tool 'maven-3.2.5'}/bin"]) {
        sh "mvn -f policyquote/pom.xml versions:set -DgenerateBackupPoms=false -DnewVersion=${version}"
    }
}

def build() {	
    withEnv(["PATH+MAVEN=${tool 'maven-3.2.5'}/bin"]) {
        sh "mvn -f policyquote/pom.xml clean package"
    }
}

def integrationTests() {
    withEnv(["PATH+MAVEN=${tool 'maven-3.2.5'}/bin"]) {	
	    sh "mvn -f policyquote/pom.xml verify"
    }
}

def publishToNexusAndCommitBranch(version, branch) {
    withEnv(["PATH+MAVEN=${tool 'maven-3.2.5'}/bin"]) {
        checkPort("nexus", "8080");
        sh "mvn -f policyquote/pom.xml deploy -DaltDeploymentRepository=internal.nexus::default::http://nexus:8080/nexus/content/repositories/releases"
        def commit = "Build " + version
        sh "git add **/pom.xml && git commit -m \"${commit}\" && git push origin ${branch}"
    }
}

def deployToBPMS(server) {
    def payload = new StringBuilder()
                        .append("<?xml version=\"1.0\" encoding=\"UTF-8\" standalone=\"yes\"?>")
                        .append("<kie-container container-id=\"policyquote-")
                        .append(env.BUILD_VERSION)
                        .append("\">")
                        .append("<release-id>")
                        .append("<group-id>")
                        .append(env.BUILD_GROUP_ID)
                        .append("</group-id>")
                        .append("<artifact-id>")
                        .append(env.BUILD_ARTIFACT_ID)
                        .append("</artifact-id>")
                        .append("<version>")
                        .append(env.BUILD_VERSION)
                        .append("</version>")
                        .append("</release-id></kie-container>")
                        .toString()

    sh "curl -X PUT -H 'Content-Type: application/xml' -d '${payload}' http://admin1:admin@${server}/kie-server/services/rest/server/containers/policyquote-${env.BUILD_VERSION}"
}


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

def String getBuildVersion(pom) {
  return getVersionFromPom(pom).minus("-SNAPSHOT") + '.' + env.BUILD_NUMBER
}
