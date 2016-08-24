def sh = { String cmd ->
	wrap([$class: 'AnsiColorBuildWrapper', 'colorMapName': 'XTerm']) {
		sh cmd
	}
}

node('windows') {
	def gopath = pwd() + "\\gopath"
	env.GOPATH = gopath
	def workdir = gopath + "\\src\\github.com\\deis\\controller-sdk-go"

	def pscmd = { String cmd ->
		"powershell -NoProfile -ExecutionPolicy Bypass -Command \"${cmd}\""
	}

	dir(workdir) {
		stage 'Checkout Windows'
			checkout scm
		stage 'Install Windows'
			bat pscmd('.\\make bootstrap')
		stage 'Test Windows'
			bat pscmd('.\\make test')
	}
}

def git_commit = ''
def git_branch = ''

stage 'Go & Git Info'
node('linux') {
	checkout scm
	// HACK: Recommended approach for getting command output is writing to and then reading a file.
	sh 'mkdir -p tmp'
	sh 'git describe --all > tmp/GIT_BRANCH'
	sh 'git rev-parse HEAD > tmp/GIT_COMMIT'
	git_branch = readFile('tmp/GIT_BRANCH').trim()
	git_commit = readFile('tmp/GIT_COMMIT').trim()

	if (git_branch != "remotes/origin/master") {
		// Determine actual PR commit, if necessary
		sh 'git rev-parse HEAD | git log --pretty=%P -n 1 --date-order > tmp/MERGE_COMMIT_PARENTS'
		sh 'cat tmp/MERGE_COMMIT_PARENTS'
		merge_commit_parents = readFile('tmp/MERGE_COMMIT_PARENTS').trim()
		if (merge_commit_parents.length() > 40) {
			echo 'More than one merge commit parent signifies that the merge commit is not the PR commit'
			echo "Changing git_commit from '${git_commit}' to '${merge_commit_parents.take(40)}'"
			git_commit = merge_commit_parents.take(40)
		} else {
			echo 'Only one merge commit parent signifies that the merge commit is also the PR commit'
			echo "Keeping git_commit as '${git_commit}'"
		}
	}
}

def test_image = "quay.io/deisci/controller-sdk-go-dev:${git_commit.take(7)}"

node('linux') {
		stage 'Build and push test container'
			checkout scm
			def quayUsername = "deisci+jenkins"
			def quayEmail = "deis+jenkins@deis.com"
			withCredentials([[$class: 'StringBinding',
												credentialsId: 'c67dc0a1-c8c4-4568-a73d-53ad8530ceeb',
									 			variable: 'QUAY_PASSWORD']]) {

				sh """
					docker login -e="${quayEmail}" -u="${quayUsername}" -p="\${QUAY_PASSWORD}" quay.io
					docker build -t ${test_image} .
					docker push ${test_image}
				"""
			}
}

stage 'Lint and test container'
parallel(
	lint: {
		node('linux') {
			sh "docker run --rm ${test_image} lint"
		}
	},
	test: {
		node('linux') {
			withCredentials([[$class: 'StringBinding',
												credentialsId: '2da033eb-2e34-4efd-b090-ad892f348065',
												variable: 'CODECOV_TOKEN']]) {
				sh "docker run -e CODECOV_TOKEN=\${CODECOV_TOKEN} --rm ${test_image} sh -c 'test-cover.sh && curl -s https://codecov.io/bash | bash'"
			}
		}
	}
)

stage 'Build and Upload CLI built with SDK'

def gcs_bucket = "gs://workflow-cli-pr"
def wcli_image = 'quay.io/deisci/workflow-cli-dev:latest'


def upload_artifacts = { String dist_dir ->
	headers  = "-h 'x-goog-meta-git-branch:${git_branch}' "
	headers += "-h 'x-goog-meta-git-sha:${git_commit}' "
	headers += "-h 'x-goog-meta-ci-job:${env.JOB_NAME}' "
	headers += "-h 'x-goog-meta-ci-number:${env.BUILD_NUMBER}' "
	headers += "-h 'x-goog-meta-ci-url:${env.BUILD_URL}'"

	script = "sh -c 'echo \${GCS_KEY_JSON} | base64 -d - > /tmp/key.json "
	script += "&& gcloud auth activate-service-account -q --key-file /tmp/key.json "
	script += "&& gsutil -mq ${headers} cp -a public-read -r /upload/* ${gcs_bucket} "
	script += "&& rm -rf /upload/*'"

	withCredentials([[$class: 'StringBinding',
										credentialsId: '6029cf4e-eaa3-4a8e-9dc7-744d118ebe6a',
										variable: 'GCSKEY']]) {
		sh "docker run ${dist_dir} -e GCS_KEY_JSON=\"\${GCSKEY}\" --rm ${wcli_image} ${script}"
	}
}

def mktmp = {
	// Create tmp directory to store files
	sh 'mktemp -d > tmp_dir'
	tmp = readFile('tmp_dir').trim()
	echo "Storing binaries in ${tmp}"
	sh 'rm tmp_dir'
	return tmp
}

node('linux') {
	flags = ""
	if (git_branch != "remotes/origin/master") {
		echo "Skipping build of 386 binaries to shorten CI for Pull Requests"
		flags += "-e BUILD_ARCH=amd64"
	}

	tmp_dir = mktmp()
	dist_dir = "-e DIST_DIR=/upload -v ${tmp_dir}:/upload"

	def pattern = "github\\.com\\/deis\\/controller-sdk-go\\n\\s+version:\\s+[a-f0-9]+"
	replacement = "github\\.com\\/deis\\/controller-sdk-go\\n"
	replacement += "  repo: https:\\/\\/github\\.com\\/${env.CHANGE_AUTHOR}\\/controller-sdk-go\\.git\\n"
	replacement += "  vcs: git\\n"
	replacement += "  version: ${git_commit}"

	build_script = "sh -c 'perl -i -0pe \"s/${pattern}/${replacement}/\" glide.yaml "
	build_script += "&& rm -rf glide.lock vendor/github.com/deis/controller-sdk-go "
	build_script += "&& glide install "
	build_script += "&& make build-revision'"
	sh "docker run ${flags} -e REVISION=${git_commit.take(7)} ${dist_dir} --rm ${wcli_image} ${build_script}"

	upload_artifacts(dist_dir)
	sh "rm -rf ${tmp_dir}"
}

stage 'Trigger e2e tests'
// If build is on master, trigger workflow-test, otherwise, assume build is a PR and trigger workflow-test-pr
waitUntil {
	try {
		def downstreamJob = git_branch == "remotes/origin/master" ? '/workflow-test' : '/workflow-test-pr'
		build job: downstreamJob, parameters: [[$class: 'StringParameterValue', name: 'WORKFLOW_CLI_SHA', value: git_commit],
			[$class: 'StringParameterValue', name: 'COMPONENT_REPO', value: 'controller-sdk-go']]
		true
	} catch(error) {
		if (git_branch == "remotes/origin/master") {
			throw error
		}

		node('linux') {
			withCredentials([[$class: 'StringBinding', credentialsId: '8a727911-596f-4057-97c2-b9e23de5268d', variable: 'SLACKEMAIL']]) {
				mail body: """<!DOCTYPE html>
<html>
<head>
<meta content='text/html; charset=UTF-8' http-equiv='Content-Type' />
</head>
<body>
<div>Author: ${env.CHANGE_AUTHOR}<br/>
Branch: ${env.BRANCH_NAME}<br/>
Commit: ${env.CHANGE_TITLE}<br/>
<p><a href="${env.BUILD_URL}console">Click here</a> to view build logs.</p>
<p><a href="${env.BUILD_URL}input/">Click here</a> to restart e2e.</p>
</div>
</html>
""", from: 'jenkins@ci.deis.io', subject: 'Controller-sdk-go E2E Test Failure', to: env.SLACKEMAIL, mimeType: 'text/html'
			}
			input "Retry the e2e tests?"
		}
		false
	}
}
