
def testReport = 'test-report.xml'
def stagingBucket = 'internal.clyqz.com'
def stagingPrefix = '/docs/whotracksme'
def productionBucket = 'cliqz-tracking-monitor'
def productionPrefix = ''

node('docker') {
    stage ('Checkout') {
        retry(5) {
            checkout scm
        }
    }
    def img

    stage('Build Docker Image') {
        img = docker.build('whotracksme', '.')
    }

    img.inside('-u 0:0') {
        try {
            stage('Install') {
                sh("python -m pip install '.[dev]'")
            }

            stage('Test') {
                try {
                    sh(script: "pytest --junit-xml=${testReport}")
                } catch(err) {
                    junit(testReport)
                    currentBuild.result = "FAILURE"
                }
            }

            stage('Build site') {
                sh('whotracksme website')
            }

            stage('Publish artifacts') {
                def tag = env.TAG_NAME
                def suffix = '-pypi'
                // Only publish a version if tag ends with `suffix`.
                if (env.BRANCH_NAME == 'master' && tag != null && tag.endsWith(suffix)) {
                    // Extract part of the tag before suffix.
                    def version = tag.substring(0, tag.length - suffix.length)

                    withCredentials([usernamePassword(
                        credentialsId: '3fc94ee4-8e89-4973-b34e-e37df209a74e',
                        passwordVariable: 'password',
                        usernameVariable: 'user'
                    )]) {
                        sh("WTM_VERSION=${version} python setup.py install")
                        sh("WTM_VERSION=${version} python setup.py sdist bdist_wheel")
                        sh('twine upload --username cliqz-oss --password $password dist/*')
                    }
                }
            }

            stage('Publish Site') {
                def deployArgs = ''
                if (env.BRANCH_NAME.contains('PR')) {
                    deployArgs = "${stagingBucket} ${stagingPrefix}/${env.BRANCH_NAME}"
                } else if (env.TAG_NAME != null) {
                    deployArgs = "${productionBucket} ${productionPrefix} --production"
                } else {
                    deployArgs = "${stagingBucket} ${stagingPrefix}/latest"
                }
                sh("python deploy_to_s3.py ${deployArgs}")
            }
        } finally {
            // cleanup
            sh('rm -rf _site; rm -rf .sass-cache')
        }
    }

    junit(testReport)
}
