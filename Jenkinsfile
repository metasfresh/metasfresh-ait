#!/usr/bin/env groovy
// the "!#/usr/bin... is just to to help IDEs, GitHub diffs, etc properly detect the language and do syntax highlighting for you.
// thx to https://github.com/jenkinsci/pipeline-examples/blob/master/docs/BEST_PRACTICES.md

// note that we set a default version for this library in jenkins, so we don't have to specify it here
@Library('misc')
import de.metas.jenkins.MvnConf
import de.metas.jenkins.Misc


// thx to http://stackoverflow.com/a/36949007/1012103 with respect to the paramters
properties([
	parameters([
		string(defaultValue: '',
			description: '''If this job is invoked via an updstream build job, than that job can provide either its branch or the respective <code>MF_UPSTREAM_BRANCH</code> that was passed to it.<br>
This build will then attempt to use maven dependencies from that branch, and it will sets its own name to reflect the given value.
<p>
So if this is a "master" build, but it was invoked by a "feature-branch" build then this build will try to get the feature-branch\'s build artifacts annd will set its
<code>currentBuild.displayname</code> and <code>currentBuild.description</code> to make it obvious that the build contains code from the feature branch.''',
			name: 'MF_UPSTREAM_BRANCH'),

		booleanParam(defaultValue: true, description: '''Set to true if this build shall trigger "endcustomer" builds.<br>
Set to false if this build is called from elsewhere and the orchestrating also takes place elsewhere''',
			name: 'MF_TRIGGER_DOWNSTREAM_BUILDS')
	]),
	pipelineTriggers([]),
	buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '20')) // keep the last 20 builds
])

final String VERSIONS_PLUGIN = 'org.codehaus.mojo:versions-maven-plugin:2.5'

timestamps
{
	MF_UPSTREAM_BRANCH = params.MF_UPSTREAM_BRANCH ?: env.BRANCH_NAME
	echo "params.MF_UPSTREAM_BRANCH=${params.MF_UPSTREAM_BRANCH}; env.BRANCH_NAME=${env.BRANCH_NAME}; => MF_UPSTREAM_BRANCH=${MF_UPSTREAM_BRANCH}"

	// https://github.com/metasfresh/metasfresh/issues/2110 make version/build infos more transparent
	final String MF_VERSION=retrieveArtifactVersion(MF_UPSTREAM_BRANCH, env.BUILD_NUMBER)
	currentBuild.displayName="artifact-version ${MF_VERSION}";

node('agent && linux') // shall only run on a jenkins agent with linux
{
	stage('Preparation') // for display purposes
	{
		// checkout our code
		checkout scm; // i hope this to do all the magic we need
		sh 'git clean -d --force -x' // clean the workspace
	}

    configFileProvider([configFile(fileId: 'metasfresh-global-maven-settings', replaceTokens: true, variable: 'MAVEN_SETTINGS')])
    {
		// create our config instance to be used further on
        final MvnConf mvnConf = new MvnConf(
					'pom.xml', // pomFile
					MAVEN_SETTINGS, // settingsFile
					"mvn-${MF_UPSTREAM_BRANCH}", // mvnRepoName
					'https://repo.metasfresh.com' // mvnRepoBaseURL
        )
        echo "mvnConf=${mvnConf}"

        nexusCreateRepoIfNotExists mvnConf.mvnDeployRepoBaseURL, mvnConf.mvnRepoName

        withMaven(jdk: 'java-8', maven: 'maven-3.5.0', mavenLocalRepo: '.repository')
        {
			  stage('Set versions and build aits')
        {

				// update the parent pom version
				mvnUpdateParentPomVersion mvnConf

				// update the metasfresh.version and metasfresh-esb-camel.version properties to the latest version (for now)
				final String metasfreshUpdatePropertyParam = '-Dproperty=metasfresh.version'
				sh "mvn --debug --settings ${mvnConf.settingsFile} --file ${mvnConf.pomFile} --batch-mode ${mvnConf.resolveParams} ${metasfreshUpdatePropertyParam} versions:update-property"

				final String metasfreshEsbCamelUpdatePropertyParam = '-Dproperty=metasfresh-esb-camel.version'
				sh "mvn --debug --settings ${mvnConf.settingsFile} --file ${mvnConf.pomFile} --batch-mode ${mvnConf.resolveParams} ${metasfreshEsbCamelUpdatePropertyParam} versions:update-property"

				// set the artifact version of everything below pom.xml
				sh "mvn --settings ${mvnConf.settingsFile} --file ${mvnConf.pomFile} --batch-mode -DallowSnapshots=false -DgenerateBackupPoms=true -DprocessDependencies=true -DprocessParent=true -DexcludeReactor=true -Dincludes=\"de.metas*:*\" ${mvnConf.resolveParams} -DnewVersion=${MF_VERSION} ${VERSIONS_PLUGIN}:set"

				// update the versions of metas dependencies that are external to the de.metas.esb reactor modules
				sh "mvn --settings ${mvnConf.settingsFile} --file ${mvnConf.pomFile} --batch-mode -DallowSnapshots=false -DgenerateBackupPoms=true -DprocessDependencies=true -DprocessParent=true -DexcludeReactor=true -Dincludes=\"de.metas*:*\" ${mvnConf.resolveParams} ${VERSIONS_PLUGIN}:use-latest-versions"

				// build and deploy
				// about -Dmetasfresh.assembly.descriptor.version: the versions plugin can't update the version of our shared assembly descriptor de.metas.assemblies. Therefore we need to provide the version from outside via this property
				// maven.test.failure.ignore=true: see metasfresh stage
				sh "mvn --settings ${mvnConf.settingsFile} --file ${mvnConf.pomFile} --batch-mode -Dmaven.test.failure.ignore=true -Dmetasfresh.assembly.descriptor.version=${MF_VERSION} ${mvnConf.resolveParams} ${mvnConf.deployParam} clean deploy"

				currentBuild.description="""artifacts (if not yet cleaned up)
				<ul>
					<li><a href=\"https://repo.metasfresh.com/content/repositories/${mvnConf.mvnRepoName}/de/metas/edi/esb/de.metas.edi.esb.camel/${MF_VERSION}/de.metas.edi.esb.camel-${MF_VERSION}.jar\">de.metas.edi.esb.camel-${MF_VERSION}.jar</a></li>
					<li><a href=\"https://repo.metasfresh.com/content/repositories/${mvnConf.mvnRepoName}/de/metas/printing/de.metas.printing.esb.base/${MF_VERSION}/de.metas.printing.esb.base-${MF_VERSION}.jar\">de.metas.printing.esb.base-${MF_VERSION}.jar</a></li>
				</ul>""";

				junit '**/target/surefire-reports/*.xml'

				jacoco()

				// gh #968:
				// set env variables which will be available to a possible upstream job that might have called us
				// all those env variables can be gotten from <buildResultInstance>.getBuildVariables()
				env.MF_VERSION="${MF_VERSION}";
      } // stage
		} // withMaven
	} // configFileProvider
 } // node
} // timestamps
