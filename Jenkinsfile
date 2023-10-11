#!/usr/bin/env groovy

/**
 * Copyright 2023 by Au-Zone Technologies.  All Rights Reserved.
 *
 * Unauthorized copying of this file, via any medium is strictly prohibited
 * Proprietary and confidential.
 */

pipeline {
    agent { label 'yocto' }

    environment {
        TEAMS_JENKINS = credentials('teams-jenkins')
        TEAMS_JENKINS_QA = credentials('teams-jenkins-qa')
        TEAMS_JENKINS_RELEASES = credentials('teams-jenkins-releases')
    }

    options {
        ansiColor('xterm')
        timeout(time: 600, unit: 'MINUTES')
        skipDefaultCheckout true

        office365ConnectorWebhooks([[
            name: 'Jenkins',
            notifyBackToNormal: true,
            notifyFailure: true,
            notifySuccess: false,
            notifyUnstable: true,
            notifyRepeatedFailure: true,
        ]])
    }

    stages {
        stage('Checkout') {
            environment {
                PATH = "${env.HOME}/bin:${env.PATH}"
            }

            steps {
                sh '''
                    if [ ! -e $HOME/bin/repo ]
                    then
                        mkdir -p $HOME/bin
                        curl https://storage.googleapis.com/git-repo-downloads/repo > $HOME/bin/repo
                        chmod +x $HOME/bin/repo
                    fi
                '''

                script {
                    if (env.TAG_NAME) {
                        REPO_BRANCH = "refs/tags/${env.TAG_NAME}"
                    } else if (env.CHANGE_BRANCH) {
                        REPO_BRANCH = env.CHANGE_BRANCH
                    } else {
                        REPO_BRANCH = env.BRANCH_NAME
                    }
                }

                sshagent(credentials : ['jenkins-ssh']) {
                    checkout scm
                    sh 'git fetch'
                    sh returnStatus: true, script: 'ssh -o StrictHostKeyChecking=no git@bitbucket.org'
                    sh returnStatus: true, script: 'ssh -o StrictHostKeyChecking=no git@github.com'
                    sh "repo init -u ssh://git@bitbucket.org/au-zone/torizon-maivin -b ${REPO_BRANCH} -m maivin/default.xml"
                    sh 'repo sync -d -j 8 --no-clone-bundle'
                }
            }
        }

        stage('Configure') {
            environment {
                MACHINE = 'verdin-imx8mp'
                EULA= '1'
            }

            steps {
                script {
                    RELEASE = env.TAG_NAME

                    if (!RELEASE) {
                        RELEASE = "${env.BRANCH_NAME.replace('/', '-')}"
                    }

                    RELEASE = RELEASE.toLowerCase()
                    IMAGE = "torizon-core-maivin"
                    TEZI = "${IMAGE}-Tezi_${RELEASE}+build.${env.BUILD_NUMBER}.tar"
                }

                sh '''#!/bin/bash
                    source ./setup-environment build
                    cp --remove-destination ../layers/meta-maivin/conf/bblayers.conf conf/bblayers.conf
                '''

                writeFile file: 'build/jenkins.conf', text:
"""
SSTATE_DIR = "${env.HOME}/yocto/sstate"
DL_DIR = "${env.HOME}/yocto/downloads"
BUILDHISTORY_COMMIT = "1"
# Disable NO_RECOMMENDATIONS until we can further test it.
NO_RECOMMENDATIONS = "0"
ACCEPT_FSL_EULA = "1"
TDX_RELEASE = "${RELEASE}"
TDX_PRERELEASE = ""
TDX_BUILDNBR = "${env.BUILD_NUMBER}"
DISTRO = "torizon-maivin"
"""
            }
        }

        stage('Download') {
            environment {
                MACHINE = 'verdin-imx8mp'
                EULA= '1'
            }

            steps {
                // Downloads can be unreliable so ensure adequate retries.
                sshagent(credentials: ['jenkins-ssh']) {
                    retry(5) {
                        sh """#!/bin/bash
                            source ./setup-environment build
                            cp --remove-destination ../layers/meta-maivin/conf/bblayers.conf conf/bblayers.conf
                            bitbake --read=jenkins.conf ${IMAGE} --runall=fetch
                        """
                    }
                }
            }
        }

        stage('Build') {
            environment {
                MACHINE = 'verdin-imx8mp'
                EULA= '1'
            }

            steps {
                sh """#!/bin/bash
                    source ./setup-environment build
                    cp --remove-destination ../layers/meta-maivin/conf/bblayers.conf conf/bblayers.conf
                    bitbake --read=jenkins.conf ${IMAGE}
                """

                sh "ln -sf build/deploy/images/$MACHINE/${TEZI}"
            }

            post {
                always {
                    jiraSendBuildInfo()
                }
            }
        }

        stage('Torizon') {
            stages {
                stage('Update Tools') {
                    steps {
                        sh 'docker pull torizon/torizoncore-builder:3'
                    }
                }

                stage('TorizonCore Builder') {
                    steps {
                        writeFile file: 'tcbuild.yaml', text:
"""
input:
  easy-installer:
    local: ${TEZI}
output:
  easy-installer:
    local: ${IMAGE}-${RELEASE}
  ostree:
    branch: ${RELEASE}
"""

                        sh '''
                            docker run --rm \
                                -v $PWD:/workdir \
                                -v storage:/storage \
                                -v /deploy \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                torizon/torizoncore-builder:3 build --force
                        '''
                    }
                }

                stage('Publish Artifacts') {
                    steps {
                        script {
                            def ostree_path = "build/tmp/work/verdin_imx8mp-tdx-linux/${IMAGE}/1.0-r0/ostree-rootfs"
                            def ostree = sh(returnStdout: true, script: "du -sb ${ostree_path} | cut -f1").trim().toInteger()
                            def tezi = findFiles(glob: "build/deploy/images/verdin-imx8mp/${TEZI}")

                            writeFile file: 'image_size.csv',
                                      text: "ostree,tezi\n${ostree / 1024 / 1024},${tezi[0].length / 1024 / 1024}"

                            plot csvFileName: 'image_size.csv',
                                csvSeries: [[ file: 'image_size.csv' ]],
                                group: 'Torizon Metrics',
                                title: 'Image Size',
                                style: 'line',
                                yaxis: 'MB'
                        }

                        dir("build/buildhistory/images/verdin_imx8mp/glibc/${IMAGE}") {
                            archiveArtifacts artifacts: 'build-id.txt'
                            archiveArtifacts artifacts: 'image-info.txt'
                            archiveArtifacts artifacts: 'installed-package-info.txt'
                        }

                        dir('build/deploy/images/verdin-imx8mp') {
                            archiveArtifacts artifacts: TEZI
                        }

                        archiveArtifacts artifacts: "${IMAGE}-${RELEASE}/**"
                        archiveArtifacts artifacts: 'tcbuild.yaml'
                    }

                    post {
                        always {
                            jiraSendDeploymentInfo environmentId: 'jenkins-tezi',
                                                   environmentName: 'Jenkins TEZI',
                                                   environmentType: 'testing'
                        }
                    }
                }

                stage('Publish Release') {
                    when { buildingTag() }

                    steps {
                        withAWS(region:'us-west-2',credentials:'maivin-s3') {
                            s3Upload bucket: 'maivin',
                                        path: "release/${env.TAG_NAME}",
                                        workingDir: "${IMAGE}-${RELEASE}",
                                        includePathPattern: '**'
                            s3Upload bucket: 'maivin',
                                        path: "release/${env.TAG_NAME}",
                                        workingDir: env.WORKSPACE,
                                        includePathPattern: '*.tar'
                        }

                        sh """
                            docker run --rm \
                                -v ${env.WORKSPACE}:/workdir \
                                -v ${env.WORKSPACE}/${IMAGE}-${RELEASE}:/workdir/image \
                                -v ${env.WORKSPACE}/splash.png:/workdir/splash.png \
                                -v $HOME/credentials.zip:/workdir/credentials.zip \
                                -v storage:/storage \
                                -v /deploy \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                torizon/torizoncore-builder:3 push \
                                    --credentials credentials.zip \
                                    --package-name torizon-maivin \
                                    --package-version ${RELEASE} \
                                    ${RELEASE}
                        """
                    }
                    post {
                        always {
                            jiraSendDeploymentInfo environmentId: 'torizon-ota',
                                                   environmentName: 'Torizon OTA',
                                                   environmentType: 'production'
                        }
                    }
                }
            }
        }
    }
}
