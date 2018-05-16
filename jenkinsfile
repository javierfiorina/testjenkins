import groovy.json.JsonOutput
import java.util.Optional
import hudson.tasks.test.AbstractTestResultAction
import hudson.model.Actionable
import hudson.tasks.junit.CaseResult
def call(body) {
    def config = [:]
    body.resolveStrategy = Closure.DELEGATE_FIRST
    body.delegate = config
    body()
    def author = ""
    def message = ""
    def testSummary = ""
    def total = 0
    def failed = 0
    def skipped = 0
    def isPublishingBranch = { ->
        return env.BRANCH_NAME == 'origin/master' || env.BRANCH_NAME =~ /release.+/
    }
    def isResultGoodForPublishing = { ->
        return currentBuild.result == null
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
    try {
        if (env.BRANCH_NAME != "master") {
            return
        }
        podTemplate(
            label: config.projectName, 
            serviceAccount: 'jenkins',
            containers: [
                    containerTemplate(name: 'jnlp', privileged: true, image: 'jenkins/jnlp-slave:alpine', args: '${computer.jnlpmac} ${computer.name}'),
                    containerTemplate(name: config.projectName, privileged: true, image: "docker-registry.default.svc:5000/openshift/${config.buildImage}", ttyEnabled: true, command: 'cat')
                ],
                volumes: [
                    persistentVolumeClaim(mountPath: '/home/jenkins/.m2', claimName: 'maven-cache', readOnly: false),
                // secretVolume(mountPath: '/home/jenkins/.maven-config', secretName: 'maven-config'),
                    secretVolume(mountPath: '/home/jenkins/.git-config', secretName: 'git-config'),
                    hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')
                ]
        ){
            node(config.projectName) {
                container(config.projectName) {
                    try {
                        stage ('Checkout') {
                            checkout scm
                        }
                        sh 'git log -1 > GIT_LOG'
                        git_log = readFile 'GIT_LOG'
                        if (git_log.contains('[image-version-update]') || git_log.contains('Merge tag')) {
                            return
                        }
                        stage('Git configuration') {
                            sh "mkdir -p /home/jenkins/.ssh"
                            sh "cp /home/jenkins/.git-config/* /home/jenkins/.ssh"
                            sh "/opt/netlabs/run.sh"
                            sh "git config --global user.email 'jenkins@netlabs.com.uy'"
                            sh "git config --global user.name 'Jenkins'"
                            sh "ssh-keyscan gitlab.netlabs.com.uy >> ~/.ssh/known_hosts"
                            sh "chmod 600 /home/jenkins/.ssh/id_rsa"
                        }
                    // stage('Maven configuration') {
                            // sh "cp /home/jenkins/.maven-config/settings.xml /home/jenkins/.m2"
                        // }
                        stage('Checkout devel') {
                            sh "git fetch"
                            sh "git checkout devel"
                            sh "git pull origin devel"
                        }
                        stage('Build') {
                            sh "mvn -Dmaven.test.skip=true package"
                        }
                        stage('Test') {
                            sh "mvn verify"
                        }
                        step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
                        stage('SonarQube Analysis') {
                            withSonarQubeEnv('Sonar') {
                                sh "mvn sonar:sonar"
                            }
                        }
                        stage("SonarQube Quality Gate") {
                            timeout(time: 1, unit: 'HOURS') {
                                qg = waitForQualityGate()
                                if (qg.status != 'OK') {
                                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                                }
                            }
                        }
                        populateGlobalVariables()
                        stage('Generate Version and Deploy') {
                            try {
                                sh "mvn -DpreparationGoals=clean -Darguments=\"-Dmaven.test.skip=true\" -B release:prepare release:perform"
                            } catch (e) {
                                sh "mvn release:rollback"
                                tag = sh(script: "git describe --tags \$(git rev-list --tags --max-count=1)", returnStdout: true).trim()
                                sh "git tag --delete ${tag}"
                                sh "git push --delete origin ${tag}"
                                throw e
                            }
                            sh "git checkout master"
                            sh "git pull origin master"
                            sh "git merge \$(git describe --tags \$(git rev-list --tags --max-count=1))"
                            sh "git push origin master"
                        }
                        def buildColor = currentBuild.result == null ? "good" : "warning"
                        def buildStatus = currentBuild.result == null ? "Success" : currentBuild.result
                        def jobName = "${env.JOB_NAME}"
                        jobName = jobName.getAt(0..(jobName.indexOf('/') - 1))
                        if (failed > 0) {
                            buildStatus = "Failed"
                            if (isPublishingBranch()) {
                                buildStatus = "MasterFailed"
                            }
                            buildColor = "danger"
                            def failedTestsString = getFailedTests()
							} else {
                            
                        }
                    } catch (hudson.AbortException ae) {
                    // I ignore aborted builds, but you're welcome to notify Slack here
                    } catch (err) {
                        def buildStatus = "Failed"
                        if (isPublishingBranch()) {
                            buildStatus = "MasterFailed"
                        }
                       
                        currentBuild.result = 'Failed'
                        throw err
                    }
                }
            }
        }
    } finally {
        // notifyBuild(currentBuild.result)
    }
}
def notifyBuild(String buildStatus = 'STARTED') {
    try {
        buildStatus =  buildStatus ?: 'SUCCESSFUL'
        def colorName = 'RED'
        def colorCode = '#FF0000'
        def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
        def summary = "${subject} (${env.BUILD_URL})"
        if (buildStatus == 'STARTED') {
            color = 'YELLOW'
            colorCode = '#FFFF00'
        } else if (buildStatus == 'SUCCESSFUL') {
            color = 'GREEN'
            colorCode = '#00FF00'
        } else {
            color = 'RED'
            colorCode = '#FF0000'
        }
        slackSend (color: colorCode, message: summary)
    } catch (err) {
        def sw = new StringWriter()
        def pw = new PrintWriter(sw)
        err.printStackTrace(pw)
        echo sw.toString()
    }
}
