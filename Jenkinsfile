IsProduction = true
IsSlackingEnabled = false
SlackChannel = null
IsEmailingEnabled = true
P4_CHANGELIST = '99999'
EmailDefault = 'user1@twgroup.org'
EmailTemplate = "groovy-html-simple.template"

// This closure is generalized to send notifications (slack msgs and/or emails) for the various build
// result states.
def SendNotifications = { BuildStatus ->
    def Subject = "Job ${BuildStatus} ${env.JOB_NAME} [${currentBuild.displayName}]"
    def Details = """<p>Jenkins Build Results Job:${env.JOB_NAME} Build:[${env.BUILD_NUMBER}]:</p>
    <p><a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>Web Page Results</p>"""
    echo "Build is: ${BuildStatus}"
    echo "Subject: ${Subject}"
    echo "Details: ${Details}"
    echo "Job URL: ${JOB_URL}"
    echo "Change List: ${P4_CHANGELIST}"
    echo "Current Build Description: ${currentBuild.description}"

    if(IsSlackingEnabled && SlackChannel != null) {
        def SlackMsg = Subject +
            "\n[<${env.JOB_URL}|Job URL>] " +
            "[<${env.BUILD_URL}|Build URL>] " +
            "[<${env.BUILD_URL}consoleText/|Full Log URL>]"
        def SlackURL = 'https://company-url.slack.com'
        def SlackSendArgs = [
            baseUrl:    "${SlackURL}/services/hooks/jenkins-ci/",
            channel:    SlackChannel,
            color:      BuildStatus == 'SUCCESS' ? 'good' :
                        BuildStatus == 'UNSTABLE' ? 'warning' :
                        BuildStatus == 'FAILURE' ? 'danger' :
                        '#439FE0',
            message:    SlackMsg,
            teamDomain: "${SlackURL}",
            tokenCredentialId: 'SlackAccessToken'
        ]
        echo "Send Slack Message to Channel ${SlackChannel}"
        //slackSend SlackSendArgs
    }

    if(IsEmailingEnabled) {
        def EmailextArgs = [
            to: "${EmailDefault}",
            subject: "${Subject}",
            body: '''${SCRIPT, template="''' + "${EmailTemplate}" + '''"}''',
            mimeType: "text/HTML",
            attachLog: false
        ]
        if(IsProduction) { EmailextArgs += [
            recipientProviders: [
            [$class: 'DevelopersRecipientProvider'],
            [$class: 'CulpritsRecipientProvider'],
            [$class: 'FirstFailingBuildSuspectsRecipientProvider'],
            [$class: 'FailingTestSuspectsRecipientProvider']
            ]
        ] }
        // Send the notification email here
        emailext EmailextArgs
    }
}

pipeline {
  agent { label 'VM-WS2019B' }
  triggers {
    pollSCM 'H/2 * * * *'
  }
  environment {
    CiGitRepoPath = 'https://github.com/techwiz52/MSBuildTest.git'
    CiBuildDir    = "C:/Development/MSBuildTest/"
    CiSlnFile     = "MSBuildTest.sln"
    CiBuildCmd    = "msbuild ${CiSlnFile} /t:build /p:Configuration=release /m"
    CiBldImage    = "mcr.microsoft.com/dotnet/framework/sdk:4.8"
    CIPublishFile = "BuildResult.zip"
  }
  stages {
    stage('Print Build Vars') {
      steps {
        echo CiBuildDir
        echo CiSlnFile
        echo "${CiBuildDir}${CiSlnFile}"
        bat 'set'
      }
    }
    stage('Sync') {
      steps { script {
        // Requires Git Plugin
        //git credentialsId: 'techwiz52', url: ${CiGitRepoPath}
        // Requires Pipeline "SCM Step" plugin (installed by default)
        // o properties and methods depend upon installed SCM plugin
        def Scm = checkout(
          [$class: 'GitSCM',
            branches: [[name: '*/master']],
            doGenerateSubmoduleConfigurations: false,
            extensions: [
              [$class: 'RelativeTargetDirectory', relativeTargetDir: "${CiBuildDir}"],
              [$class: 'CleanBeforeCheckout']
            ],
            submoduleCfg: [],
            userRemoteConfigs: [
              [credentialsId: 'techwiz52st',
                name: 'origin',
                url: "${CiGitRepoPath}"
              ]
            ]
          ]
        )
      echo "ScmVars=${Scm}"
      } }
    }
    stage('Build') {
      steps {
        echo "docker run --rm -w ${CiBuildDir} -v ${CiBuildDir}:${CiBuildDir} ${CiBldImage} ${CiBuildCmd}"
        bat "docker run --rm -w ${CiBuildDir} -v ${CiBuildDir}:${CiBuildDir} ${CiBldImage} ${CiBuildCmd}"
      }
    }
    stage('Publish') {
      steps {
        dir("${CiBuildDir}") {
          // Requires File Operations plugin
          fileOperations(
            [fileDeleteOperation(excludes: '', includes: "${CIPublishFile}")]
          )
          // Requires Pipeline Utilities Step plugin
          zip zipFile: "${CIPublishFile}", archive: false, dir: 'bin', glob: '*.*'
          // Requires Artifact Archiver plug (installed default)
          archiveArtifacts "${CIPublishFile}"
        }
      }
    }
  }
  post{
      failure { script {
          currentBuild.result = 'FAILURE'
          def BuildStatus =  currentBuild.result.toString()
          SendNotifications(BuildStatus)
      } }
      unstable { script {
          currentBuild.result = 'UNSTABLE'
          def BuildStatus =  currentBuild.result.toString()
          SendNotifications(BuildStatus)
      } }
      fixed { script {
          currentBuild.result = 'SUCCESS'
          def BuildStatus =  currentBuild.result.toString()
          SendNotifications(BuildStatus)
      } }
  }
}
