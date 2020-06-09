pipeline {
    agent any

    options() {
        disableConcurrentBuilds()
    }

    parameters {
        booleanParam(defaultValue: false, description: 'Skal prosjektet releases?', name: 'isRelease')
        string(name: "releaseVersion", defaultValue: "", description: "Hva er det nye versjonsnummeret?")
        string(name: "snapshotVersion", defaultValue: "", description: "Hva er den nye snapshotversjonen? (uten -SNAPSHOT postfix)")
    }

    stages {

        stage('Resolve version') {
            steps {
                script {
                    env.IMAGE_NAME = 'fiks-nginx-openshift'
                    env.GIT_SHA = sh(returnStdout: true, script: 'git rev-parse HEAD').substring(0, 7)
                    env.WORKSPACE = pwd()
                    env.CURRENT_VERSION = readFile "${env.WORKSPACE}/version"
                    env.CURRENT_VERSION = env.CURRENT_VERSION.replace("SNAPSHOT", env.GIT_SHA)
                    env.IMAGE_TAG = env.CURRENT_VERSION
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
                expression { params.isRelease }
            }

            steps {
                script {
                    if (params.releaseVersion == null || params.releaseVersion == "" || params.snapshotVersion == null || params.snapshotVersion == ""){
                        currentBuild.result = 'ABORTED'
                        error("release and snapshot version must be set")
                    }

                    env.CURRENT_VERSION = params.releaseVersion
                    env.IMAGE_TAG = env.CURRENT_VERSION
                    currentBuild.description = "Release: ${params.releaseVersion}"
                }

                gitCheckout()

                writeFile(file: "${env.WORKSPACE}/version", text: params.releaseVersion);

                sh 'git add version'
                sh 'git commit -m "new release version"'
                sh "git tag -a ${params.releaseVersion} -m \"Releasing jenkins build ${env.BUILD_NUMBER}\""
                gitPush()
            }
        }
        
        stage("push images") {
            steps {
                script {
                    buildAndPushDockerImage(env.IMAGE_NAME, [env.CURRENT_VERSION, 'latest'], [], params.isRelease, "nginx")
                }
            } 
        }
        

        stage('Release: Set new snapshot version') {
            when {
                expression { params.isRelease }
            }

            steps {
                writeFile(file: "${env.WORKSPACE}/version", text: "${params.snapshotVersion}-SNAPSHOT");
                sh 'git add version'
                sh "git commit -m \"Setting new snapshot version to ${params.snapshotVersion}-SNAPSHOT\""
                gitPush()
            }
        }
    }
}