#!groovy

import groovy.json.JsonOutput
import java.util.Optional
import hudson.tasks.test.AbstractTestResultAction
import hudson.model.Actionable
import hudson.tasks.junit.CaseResult

def speedUp = '--configure-on-demand --daemon --parallel'
def nebulaReleaseScope = (env.GIT_BRANCH == 'origin/master') ? '' : "-Prelease.scope=patch"
def nebulaRelease = "-x prepare -x release snapshot ${nebulaReleaseScope}"
def gradleDefaultSwitches = "${speedUp} ${nebulaRelease}"
def gradleAdditionalTestTargets = "integrationTest"
def gradleAdditionalSwitches = "shadowJar"
def slackNotificationChannel = "#alerts"
def author = ""
def message = ""
def testSummary = ""
def total = 0
def failed = 0
def skipped = 0

def isPublishingBranch = { ->
    return env.GIT_BRANCH == 'origin/master' || env.GIT_BRANCH =~ /release.+/
}

def isResultGoodForPublishing = { ->
    return currentBuild.result == null
}

def notifySlack(text, channel, attachments) {
    def slackURL = 'https://hooks.slack.com/services/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'
    def jenkinsIcon = 'https://wiki.jenkins-ci.org/download/attachments/2916393/logo.png'

    def payload = JsonOutput.toJson([text: text,
        channel: channel,
        username: "Jenkins",
        icon_url: jenkinsIcon,
        attachments: attachments
    ])

    sh "curl -X POST --data-urlencode \'payload=${payload}\' ${slackURL}"
}

def getGitAuthor = {
    def commit = sh(returnStdout: true, script: 'git rev-parse HEAD')
    author = sh(returnStdout: true, script: "git --no-pager show -s --format='%an' ${commit}").trim()
}

def getLastCommitMessage = {
    message = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()
}

@NonCPS
def getTestSummary = { ->
    def testResultAction = currentBuild.rawBuild.getAction(AbstractTestResultAction.class)
    def summary = ""

    if (testResultAction != null) {
        total = testResultAction.getTotalCount()
        failed = testResultAction.getFailCount()
        skipped = testResultAction.getSkipCount()

        summary = "Passed: " + (total - failed - skipped)
        summary = summary + (", Failed: " + failed)
        summary = summary + (", Skipped: " + skipped)
    } else {
        summary = "No tests found"
    }
    return summary
}

@NonCPS
def getFailedTests = { ->
    def testResultAction = currentBuild.rawBuild.getAction(AbstractTestResultAction.class)
    def failedTestsString = "```"

    if (testResultAction != null) {
        def failedTests = testResultAction.getFailedTests()

        if (failedTests.size() > 9) {
            failedTests = failedTests.subList(0, 8)
        }

        for(CaseResult cr : failedTests) {
            failedTestsString = failedTestsString + "${cr.getFullDisplayName()}:\n${cr.getErrorDetails()}\n\n"
        }
        failedTestsString = failedTestsString + "```"
    }
    return failedTestsString
}

def populateGlobalVariables = {
    getLastCommitMessage()
    getGitAuthor()
    testSummary = getTestSummary()
}

def label = "mypod-${UUID.randomUUID().toString()}"
def project = "alert-inquiry-205619"

podTemplate(label: label, containers: [
    containerTemplate(name: 'gcloud', image: 'google/cloud-sdk:latest', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'sbt', image: 'hseeberger/scala-sbt:8u181_2.12.7_1.2.6', ttyEnabled: true, command: 'cat')
]) {
       
node ("scala_agent") {
    try {
        stage('Checkout') {
            checkout scm
        }
    
        container('sbt') {
        stage('Build') {
            sh('sbt sbtVersion')
            sh('cd bowling-kata;')
            sh('sbt clean assembly')
            sh('ls -al ~/')
            archiveArtifacts artifacts: 'target/**/*.jar', fingerprint: true
            //sh "sbt clean assembly"
            //step $class: 'JUnitResultArchiver', testResults: '**/TEST-*.xml'

            populateGlobalVariables()

            def buildColor = currentBuild.result == null ? "good" : "warning"
            def buildStatus = currentBuild.result == null ? "Success" : currentBuild.result
            def jobName = "${env.JOB_NAME}"

            // Strip the branch name out of the job name (ex: "Job Name/branch1" -> "Job Name")
            jobName = jobName.getAt(0..(jobName.indexOf('/') - 1))

            if (failed > 0) {
                buildStatus = "Failed"

                if (isPublishingBranch()) {
                    buildStatus = "MasterFailed"
                }

                buildColor = "danger"
                def failedTestsString = getFailedTests()

                notifySlack("", slackNotificationChannel, [
                    [
                        title: "${jobName}, build #${env.BUILD_NUMBER}",
                        title_link: "${env.BUILD_URL}",
                        color: "${buildColor}",
                        text: "${buildStatus}\n${author}",
                        "mrkdwn_in": ["fields"],
                        fields: [
                            [
                                title: "Branch",
                                value: "${env.GIT_BRANCH}",
                                short: true
                            ],
                            [
                                title: "Test Results",
                                value: "${testSummary}",
                                short: true
                            ],
                            [
                                title: "Last Commit",
                                value: "${message}",
                                short: false
                            ]
                        ]
                    ],
                    [
                        title: "Failed Tests",
                        color: "${buildColor}",
                        text: "${failedTestsString}",
                        "mrkdwn_in": ["text"],
                    ]
                ])
            } else {
                notifySlack("", slackNotificationChannel, [
                    [
                        title: "${jobName}, build #${env.BUILD_NUMBER}",
                        title_link: "${env.BUILD_URL}",
                        color: "${buildColor}",
                        author_name: "${author}",
                        text: "${buildStatus}\n${author}",
                        fields: [
                            [
                                title: "Branch",
                                value: "${env.GIT_BRANCH}",
                                short: true
                            ],
                            [
                                title: "Test Results",
                                value: "${testSummary}",
                                short: true
                            ],
                            [
                                title: "Last Commit",
                                value: "${message}",
                                short: false
                            ]
                        ]
                    ]
                ])
            }
        }
        }    

        if (isPublishingBranch() && isResultGoodForPublishing()) {
            stage ('Publish') {
                sh "./gradlew ${gradleDefaultSwitches}"
            }
        }
    } catch (hudson.AbortException ae) {
        // I ignore aborted builds, but you're welcome to notify Slack here
    } catch (e) {
        def buildStatus = "Failed"

        if (isPublishingBranch()) {
            buildStatus = "MasterFailed"
        }

        notifySlack("", slackNotificationChannel, [
            [
                title: "${env.JOB_NAME}, build #${env.BUILD_NUMBER}",
                title_link: "${env.BUILD_URL}",
                color: "danger",
                author_name: "${author}",
                text: "${buildStatus}",
                fields: [
                    [
                        title: "Branch",
                        value: "${env.GIT_BRANCH}",
                        short: true
                    ],
                    [
                        title: "Test Results",
                        value: "${testSummary}",
                        short: true
                    ],
                    [
                        title: "Last Commit",
                        value: "${message}",
                        short: false
                    ],
                    [
                        title: "Error",
                        value: "${e}",
                        short: false
                    ]
                ]
            ]
        ])

        throw e
    }
    
}
}
