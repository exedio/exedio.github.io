
timestamps
{
	def jdk = 'openjdk-11'

	//noinspection GroovyAssignabilityCheck
	node('GitCloneExedio && docker')
	{
		try
		{
			abortable
			{
				echo("Delete working dir before build")
				deleteDir()

				checkout scm

				properties([
						buildDiscarder(logRotator(
								numToKeepStr         : '1000',
								artifactNumToKeepStr : '1000'
						))
				])

				def dockerName = env.JOB_NAME.replace("/", "-") + "-" + env.BUILD_NUMBER
				def dockerDate = new Date().format("yyyyMMdd")
				def mainImage = docker.build(
						'exedio-jenkins:' + dockerName + '-' + dockerDate,
						'--build-arg JDK=' + jdk + ' ' +
						'conf/main')
				mainImage.inside(
						"--name '" + dockerName + "' " +
						"--cap-drop all " +
						"--security-opt no-new-privileges " +
						"--network none")
				{
					sh "ant -noinput clean jenkins"
				}

				recordIssues(
						failOnError: true,
						enabledForFailure: true,
						ignoreFailedBuilds: false,
						qualityGates: [[threshold: 1, type: 'TOTAL', unstable: true]],
						tools: [
							java(),
						],
				)

				sh "git status --porcelain --untracked-files=normal > git-status.txt"
				def gitStatus = readFile('git-status.txt')
				if(gitStatus!=' M api/overview-summary.html\n?? git-status.txt\n')
				{
					archive 'git-status.txt'
					currentBuild.result = 'FAILURE';
				}
			}
		}
		catch(Exception e)
		{
			//todo handle script returned exit code 143
			throw e;
		}
		finally
		{
			def to = emailextrecipients([
					[$class: 'CulpritsRecipientProvider'],
					[$class: 'RequesterRecipientProvider']
			])
			//TODO details
			step([$class: 'Mailer',
					recipients: to,
					attachLog: true,
					notifyEveryUnstableBuild: true])

			echo("Delete working dir after " + currentBuild.result)
			deleteDir()
		}
	}
}

def abortable(Closure body)
{
	try
	{
		body.call();
	}
	catch(hudson.AbortException e)
	{
		if(e.getMessage().contains("exit code 143"))
			return
		throw e;
	}
}
