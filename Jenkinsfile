#!groovy

properties([
  // Keep only the most recent builds in order to preserve space
  buildDiscarder(logRotator(numToKeepStr: '20')),
  // Don't run concurrent builds for a branch, because they use the same workspace directory
  disableConcurrentBuilds()
])

node {

  def library
  def mvn
  def git

  catchError {

    stage('Checkout') {
      checkout scm

      library = loadLocalLib('ces-build-lib').com.cloudogu.ces.cesbuildlib
      mvn = library.MavenWrapper.new(this)
      git = library.Git.new(this)

      git.clean('')
    }

    initMaven(mvn)

    stage('Build') {
      mvn 'clean install -DskipTests'
      archive '**/target/*.jar'
    }

    stage('Unit Test') {
      mvn 'test'
    }

    stage('Integration Test') {
      mvn 'verify -DskipUnitTests'
    }

    stage('Static Code Analysis') {
      def sonarQube = library.SonarCloud.new(this, [sonarQubeEnv: 'sonarcloud.io-cloudogu'])

      sonarQube.analyzeWith(mvn)

      if (!sonarQube.waitForQualityGateWebhookToBeCalled()) {
        currentBuild.result ='UNSTABLE'
      }
    }

    stage('Deploy') {
      if (preconditionsForDeploymentFulfilled()) {

        mvn.useDeploymentRepository([id: 'ossrh', url: 'https://oss.sonatype.org/',
                                     credentialsId: 'mavenCentral-acccessToken', type: 'Nexus2'])

        mvn.setSignatureCredentials('mavenCentral-secretKey-asc-file',
                                    'mavenCentral-secretKey-Passphrase')

        mvn.deployToNexusRepositoryWithStaging()
      }
    }
  }

  // Archive Unit and integration test results, if any
  junit allowEmptyResults: true,
    testResults: '**/target/surefire-reports/TEST-*.xml, **/target/failsafe-reports/*.xml'

  // Find maven warnings and visualize in job
  warnings consoleParsers: [[parserName: 'Maven']]

  mailIfStatusChanged(git.commitAuthorEmail)
}

void loadLocalLib(String path) {
  // create new git repo inside jenkins subdirectory
  dir(path) {
    sh('git init && git config user.name "Your Name" && git config user.email "you@example.com" && git add --all . && git commit -m init &> /dev/null')
    library identifier: 'local-lib@master', retriever: modernSCM([$class: 'GitSCMSource', remote: pwd()]), changelog: false
  }
}

boolean preconditionsForDeploymentFulfilled() {
  if (isBuildSuccessful() &&
      !isPullRequest() &&
      shouldBranchBeDeployed()) {
    return true
  } else {
    echo "Skipping deployment because of branch or build result: currentResult=${currentBuild.currentResult}, " +
      "result=${currentBuild.result}, branch=${env.BRANCH_NAME}."
    return false
  }
}

private boolean shouldBranchBeDeployed() {
  return env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'develop'
}

private boolean isBuildSuccessful() {
  currentBuild.currentResult == 'SUCCESS' &&
    // Build result == SUCCESS seems not to set be during pipeline execution.
    (currentBuild.result == null || currentBuild.result == 'SUCCESS')
}

void initMaven(mvn) {

  if ("master".equals(env.BRANCH_NAME)) {

    echo "Building master branch"
    mvn.additionalArgs = "-DperformRelease"
    currentBuild.description = mvn.getVersion()
  }
}