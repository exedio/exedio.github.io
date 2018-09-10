
timestamps
{
	//noinspection GroovyAssignabilityCheck
	node('GitCloneExedio && OpenJdk18Debian9')
	{
		try
		{
			abortable
			{
				echo("Delete working dir before build")
				deleteDir()

				checkout scm

				env.JAVA_HOME = "${tool 'openjdk 1.8 debian9'}"
				env.PATH = "${env.JAVA_HOME}/bin:${env.PATH}"
				def antHome = tool 'Ant version 1.9.3'

				properties([
						buildDiscarder(logRotator(
								numToKeepStr         : '1000',
								artifactNumToKeepStr : '1000'
						))
				])

				sh "${antHome}/bin/ant clean jenkins"

				warnings(
						canComputeNew: true,
						canResolveRelativePaths: true,
						categoriesPattern: '',
						consoleParsers: [[parserName: 'Java Compiler (javac)']],
						defaultEncoding: '', excludePattern: '', healthy: '', includePattern: '', messagesPattern: '', unHealthy: '',
						unstableTotalAll: '0',
						usePreviousBuildAsReference: false,
						useStableBuildAsReference: false,
				)

				sh "git status --porcelain --untracked-files=normal > git-status.txt"
				def gitStatus = readFile('git-status.txt')
				if(gitStatus!='?? git-status.txt\n')
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
