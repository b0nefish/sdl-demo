def notify(status){
	emailext (
		body: '$DEFAULT_CONTENT', 
		recipientProviders: [
			[$class: 'CulpritsRecipientProvider'],
			[$class: 'DevelopersRecipientProvider'],
			[$class: 'RequesterRecipientProvider']
		], 
		replyTo: '$DEFAULT_REPLYTO', 
		subject: '$DEFAULT_SUBJECT',
		to: '$DEFAULT_RECIPIENTS'
	)
}

def buildStep(ext) {
	sh "rm -rfv build-$ext"
	sh "mkdir -p build-$ext"
	sh "cd build-$ext && cmake -DCMAKE_TOOLCHAIN_FILE=/opt/cmake$ext .."
	sh "cd build-$ext && make -j8"

	if (!env.CHANGE_ID) {
		sh "mv build-$ext/sdl-demo publishing/deploy/sdl-demo/sdl-demo.$ext"
		//sh "cp publishing/amiga-spec/sdl-demo.info publishing/deploy/sdl-demo/sdl-demo.$ext.info"
	}
}

node {
	try{
		stage('Checkout and pull') {
			slackSend color: "good", message: "Build Started: ${env.JOB_NAME} #${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"

			properties([pipelineTriggers([githubPush()])])
			if (env.CHANGE_ID) {
				echo 'Trying to build pull request'
			}

			checkout scm
		}
	
		if (!env.CHANGE_ID) {
			stage('Generate publishing directories') {
				sh "rm -rfv publishing/deploy/*"
				sh "mkdir -p publishing/deploy/sdl-demo"
			}
		}

		stage('Build WarpOS version') {
			buildStep('wos')
		}

		stage('Build AmigaOS 3.x version') {
			buildStep('68k')
		}

		stage('Build AmigaOS 4.x version') {
			buildStep('os4')
		}

		stage('Build MorphOS 3.x version') {
			buildStep('mos')
		}

		stage('Build AROS x86 ABI-v1 version') {
			buildStep('aros-abiv1-x86')
		}

		stage('Build AROS x86_64 ABI-v1 version') {
			buildStep('aros-abiv1-x86_64')
		}

		stage('Deploying to stage') {
			if (env.TAG_NAME) {
				sh "echo $TAG_NAME > publishing/deploy/STABLE"
				sh "ssh $DEPLOYHOST mkdir -p public_html/downloads/releases/sdl-demo/$TAG_NAME"
				sh "scp publishing/deploy/sdl-demo/* $DEPLOYHOST:~/public_html/downloads/releases/sdl-demo/$TAG_NAME/"
				sh "scp publishing/deploy/STABLE $DEPLOYHOST:~/public_html/downloads/releases/sdl-demo/"
			} else if (env.BRANCH_NAME.equals('master')) {
				sh "date +'%Y-%m-%d %H:%M:%S' > publishing/deploy/BUILDTIME"
				sh "ssh $DEPLOYHOST mkdir -p public_html/downloads/nightly/sdl-demo/`date +'%Y'`/`date +'%m'`/`date +'%d'`/"
				sh "scp publishing/deploy/sdl-demo/* $DEPLOYHOST:~/public_html/downloads/nightly/sdl-demo/`date +'%Y'`/`date +'%m'`/`date +'%d'`/"
				sh "scp publishing/deploy/BUILDTIME $DEPLOYHOST:~/public_html/downloads/nightly/sdl-demo/"
			}
			slackSend color: "good", message: "Build Succeeded: ${env.JOB_NAME}"
		}
	
	} catch(err) {
		currentBuild.result = 'FAILURE'
		slackSend color: "danger", message: "Build Failed: ${env.JOB_NAME}"
		notify('Build failed')
		throw err
	}
}

