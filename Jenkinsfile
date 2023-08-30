#!'groovy'
import groovy.transform.Field
import groovy.transform.stc.ClosureParams
import groovy.transform.stc.SimpleType

@Field
String jdk = 'openjdk-11'

boolean isRelease = env.BRANCH_NAME=="master"

Map<String, ?> recordIssuesDefaults = [
	failOnError         : true,
	enabledForFailure   : true,
	ignoreFailedBuilds  : false,
	skipPublishingChecks: true,
	qualityGates        : [[threshold: 1, type: 'TOTAL', unstable: true]],
]

properties([
		gitLabConnection(env.GITLAB_CONNECTION),
		buildDiscarder(
			logRotator(
				numToKeepStr         : isRelease ? '1000' : '30',
				artifactNumToKeepStr : isRelease ?  '100' :  '2',
		))
])

boolean tryCompleted = false
try
{
	Map<String, Closure<?>> parallelBranches = [:]

	parallelBranches["Main"] = {
		nodeCheckoutAndDelete { scmResult ->

			mainImage(imageName("Main")).inside(dockerRunDefaults()) {
				ant 'clean jenkins'
			}

			recordIssues(
				*: recordIssuesDefaults,
				tools: [
					java(),
				],
			)

			assertGitUnchanged()
		}
	}

	parallel parallelBranches

	tryCompleted = true
}
finally
{
	if (!tryCompleted)
		currentBuild.result = 'FAILURE'

	// workaround for Mailer plugin: set result status explicitly to SUCCESS if empty, otherwise no mail will be triggered if a build gets successful after a previous unsuccessful build
	if (currentBuild.result == null)
		currentBuild.result = 'SUCCESS'
	node('email') {
		step(
			[$class                  : 'Mailer',
			 recipients              : emailextrecipients([isRelease ? culprits() : developers(), requestor()]),
			 notifyEveryUnstableBuild: true])
	}
	// https://docs.gitlab.com/ee/api/commits.html#post-the-build-status-to-a-commit
	updateGitlabCommitStatus state: currentBuild.resultIsBetterOrEqualTo("SUCCESS") ? "success" : "failed"
}

// ------------------- LIBRARY ----------------------------
// The code below is meant to be equal across all projects.

void nodeCheckoutAndDelete(@ClosureParams(value = SimpleType, options = ["Map<String, String>"]) Closure body)
{
	node('GitCloneExedio && docker') {
		env.JENKINS_OWNER = shStdout("id --user") + ':' + shStdout("id --group")
		try
		{
			deleteDir()
			def scmResult = checkout scm
			updateGitlabCommitStatus state: 'running'

			body.call(scmResult)
		}
		finally
		{
			deleteDir()
		}
	}
}

String jobNameAndBuildNumber()
{
	env.JOB_NAME.replace("/", "-").replace(" ", "_") + "-" + env.BUILD_NUMBER
}

def mainImage(String imageName)
{
	return docker.build(
		imageName,
		'--build-arg JDK=' + jdk + ' ' +
		'--build-arg JENKINS_OWNER=' + env.JENKINS_OWNER + ' ' +
		'conf/main')
}

String imageName(String pipelineBranch, String subImage = '')
{
	String isoToday = new Date().format("yyyyMMdd")
	String name = 'exedio-jenkins:' + jobNameAndBuildNumber() + '-' + pipelineBranch + '-' + isoToday
	if (!subImage.isBlank()) name += '-' + subImage
	return name
}

static String dockerRunDefaults(String network = 'none', String hostname = '')
{
	return "--cap-drop all " +
	       "--security-opt no-new-privileges " +
	       "--network " + network + " " +
	       (hostname != '' ? "--network-alias " + hostname + " " : "") +
	       "--dns-opt timeout:1 " + // seconds; default is 5
	       "--dns-opt attempts:1 " // default is 2
}

void shSilent(String script)
{
	try
	{
		sh script
	}
	catch (Exception ignored)
	{
		currentBuild.result = 'FAILURE'
	}
}

String shStdout(String script)
{
	return ((String) sh(script: script, returnStdout: true)).trim()
}

void ant(String script)
{
	shSilent 'java -jar submodules/persistence/ant/lib/ant-launcher.jar -noinput ' + script
}

void assertGitUnchanged()
{
	String gitStatus = shStdout "git status --porcelain --untracked-files=normal"
	if (gitStatus != '')
	{
		error 'FAILURE because fetching dependencies produces git diff:\n' + gitStatus
	}
}
