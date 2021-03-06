if (env.BRANCH_NAME == 'master') {
    properties([
        pipelineTriggers([
            triggers: [
                [
                    $class: 'jenkins.triggers.ReverseBuildTrigger',
                    upstreamProjects: 'ocflib-upload-pypi', threshold: hudson.model.Result.SUCCESS
                ]
            ]
        ]),
    ])
}

node('slave') {
    step([$class: 'WsCleanup'])

    stage('check-out-code') {
        // TODO: I think we can factor out the "src" subdirectory now that we
        // don't build a Debian package?
        dir('src') {
            checkout scm

            // TODO: figure out how to get the git plugin to do this for us
            sh 'git submodule update --init'
        }
    }

    stage('test') {
        dir('src') {
            sh 'make test'
        }
    }

    stash 'src'
}


if (env.BRANCH_NAME == 'master') {
    def version = new Date().format("yyyy-MM-dd-'T'HH-mm-ss")
    withEnv([
        'DOCKER_REPO=docker-push.ocf.berkeley.edu/',
        "DOCKER_REVISION=${version}",
    ]) {
        node('slave') {
            step([$class: 'WsCleanup'])
            unstash 'src'

            stage('cook-prod-image') {
                dir('src') {
                    sh 'make cook-image'
                }
            }

            stash 'src'
        }

        node('deploy') {
            step([$class: 'WsCleanup'])
            unstash 'src'

            stage('push-to-registry') {
                dir('src') {
                    sh 'make push-image'
                }
            }

            stage('deploy-to-prod') {
                // TODO: make these deploy and roll back together!
                build job: 'marathon-deploy-app', parameters: [
                    [$class: 'StringParameterValue', name: 'app', value: 'ocfweb/web'],
                    [$class: 'StringParameterValue', name: 'version', value: version],
                ]
                build job: 'marathon-deploy-app', parameters: [
                    [$class: 'StringParameterValue', name: 'app', value: 'ocfweb/worker'],
                    [$class: 'StringParameterValue', name: 'version', value: version],
                ]
                build job: 'marathon-deploy-app', parameters: [
                    [$class: 'StringParameterValue', name: 'app', value: 'ocfweb/static'],
                    [$class: 'StringParameterValue', name: 'version', value: version],
                ]
            }
        }
    }
} else {
    // TODO: figure out how to do this in parallel with the previous step
    node('slave') {
        step([$class: 'WsCleanup'])
        unstash 'src'

        stage('test-cook-image') {
            dir('src') {
                sh 'make cook-image'
            }
        }
    }
}


// vim: ft=groovy
