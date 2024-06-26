pipeline {
    agent any

    options() {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20'))
    }

    parameters {
        booleanParam(defaultValue: false, description: 'Skal prosjektet releases?', name: 'isRelease')
    }
    environment {
        RELEASE_VERSION = readFile("version").trim()
        NEXT_VERSION = sh(script: "./increment_version.sh -p ${RELEASE_VERSION}", returnStdout: true).trim()
        user = buildUser()
    }

    stages {

        stage('Resolve version') {
            steps {
                script {
                    env.IMAGE_NAME = 'fiks-nginx-openshift'
                    env.GIT_SHA = sh(returnStdout: true, script: 'git rev-parse HEAD').substring(0, 7)
                    env.WORKSPACE = pwd()
                    env.CURRENT_VERSION = chartVersion(RELEASE_VERSION, GIT_BRANCH, BUILD_NUMBER, params.isRelease) 
                    env.IMAGE_TAG = env.CURRENT_VERSION
                    env.REPO_NAME = scm.getUserRemoteConfigs()[0].getUrl().tokenize('/').last().split("\\.")[0]
                }

            }
        }
        stage("build image") {       
            steps {
                script {
                    buildDockerImage(env.IMAGE_NAME, [env.CURRENT_VERSION, 'latest'], [], "nginx")
                }
            }
        }
        stage('Release: Set new release version') {
            when {
                allOf {
                    expression { params.isRelease }
                    branch 'main'
                }
            }

            steps {
                gitCheckout("main")
                sh(script: "git tag -f -a ${env.RELEASE_VERSION} -m \"Releasing jenkins build ${env.BUILD_NUMBER}\"", label: "Tagging release ${env.RELEASE_VERSION}")
                writeFile(file: "${env.WORKSPACE}/version", text: env.NEXT_VERSION);
                sh(script: 'git add version && git commit -a -m "Prepare further development"', label: "Commit oppdatert versjonsfil")
            }
            post {
                success {
                    gitPush()
                    script {
                        currentBuild.description = "${env.user} released version ${env.RELEASE_VERSION}"
                    }
                }
                
            }
        }
        stage("push images ") { 
            parallel {
                stage("Publish to internal repo") {
                    steps {
                        script {
                            buildAndPushDockerImage(env.IMAGE_NAME, [env.CURRENT_VERSION, 'latest'], [], params.isRelease, "nginx")
                        }
                        withDockerRegistry(credentialsId: 'artifactory-token-based', url: 'https://docker-local-snapshots.artifactory.fiks.ks.no') {
                            catchError {
                                sh script: "docker sbom --format cyclonedx-json -o ${IMAGE_NAME}-sbom.json docker-local-snapshots.artifactory.fiks.ks.no/${IMAGE_NAME}:${CURRENT_VERSION}"
                            }
                        }
                    }
                    post {
                        success {
                            archiveArtifacts artifacts: "*-sbom.json", fingerprint: true, allowEmptyArchive: true
                            publishDependencyTrack("2a2f37ae-e189-4e28-b434-8866f86346b3", env.IMAGE_NAME, env.CURRENT_VERSION, "${IMAGE_NAME}-sbom.json")
                        }
                    }                     
                }

                stage("Publish to Github Packages") {
                    when {
                        anyOf{
                            expression { params.isRelease }
                        }
                    }
                    steps {
                        script {
                            buildAndPushDockerImageGH(env.IMAGE_NAME, [env.CURRENT_VERSION, 'latest'], [], "nginx")
                        }
                    }
                }
            }
        }
    }
    post {
       always {
            jiraSendBuildInfo(
                site: "ksfiks.atlassian.net"
            )
            deleteDir()
        }
    }
}

def buildUser() {
    wrap([$class: 'BuildUser']) {
        return sh(script: 'echo "${BUILD_USER}"', returnStdout: true).trim()                
    }
}