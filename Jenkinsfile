
timestamps
{
	def jdk = 'openjdk-11-deb10'

	//noinspection GroovyAssignabilityCheck
	node('GitCloneExedio && ' + jdk)
	{
		try
		{
			abortable
			{
				echo("Delete working dir before build")
				deleteDir()

				checkout scm

				env.JAVA_HOME = tool jdk
				env.PATH = "${env.JAVA_HOME}/bin:${env.PATH}"
				def antHome = tool 'Ant version 1.9.3'

				properties([
						buildDiscarder(logRotator(
								numToKeepStr         : '1000',
								artifactNumToKeepStr : '1000'
						))
				])

				sh "${antHome}/bin/ant clean jenkins"

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
