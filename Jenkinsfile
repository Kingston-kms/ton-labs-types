G_giturl = ""
G_gitcred = 'TonJenSSH'
G_docker_creds = "TonJenDockerHub"
G_params = null
G_images = [:]
G_docker_image = null
G_commit = ""
G_binversion = "NotSet"

def isUpstream() {
    return currentBuild.getBuildCauses()[0]._class.toString() == 'hudson.model.Cause$UpstreamCause'
}

pipeline {
    tools {nodejs "Node12.8.0"}
    options {
        buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '1')
        
        parallelsAlwaysFailFast()
    }
    agent {
        node {
            label 'master'
        }
    }
    parameters {
        string(
            name:'common_version',
            defaultValue: '',
            description: 'Common version'
        )
        string(
            name:'image_ton_labs_types',
            defaultValue: '',
            description: 'ton-labs-types image name'
        )
    }
    stages {
        stage('Collect commit data') {
            steps {
                sshagent([G_gitcred]) {
                    script {
                        G_giturl = env.GIT_URL
                        echo "${G_giturl}"
                        if(isUpstream() && GIT_BRANCH != "master") {
                            checkout(
                                [$class: 'GitSCM', 
                                branches: [[name: "origin/${GIT_BRANCH}"]], 
                                doGenerateSubmoduleConfigurations: false, 
                                extensions: [[
                                    $class: 'PreBuildMerge', 
                                    options: [
                                        mergeRemote: 'origin', 
                                        mergeTarget: 'master'
                                    ]
                                ]], 
                                submoduleCfg: [], 
                                userRemoteConfigs: [[credentialsId: 'TonJen', url: G_giturl]]])
                            G_commit = sh (script: 'git rev-parse HEAD^{commit}', returnStdout: true).trim()
                            echo "${GIT_COMMIT} merged with master. New commit ${G_commit}"
                        } else {
                            G_commit = GIT_COMMIT
                        }
                        C_PROJECT = env.GIT_URL.substring(19, env.GIT_URL.length() - 4)
                        C_COMMITER = sh (script: 'git show -s --format=%cn ${GIT_COMMIT}', returnStdout: true).trim()
                        C_TEXT = sh (script: 'git show -s --format=%s ${GIT_COMMIT}', returnStdout: true).trim()
                        C_AUTHOR = sh (script: 'git show -s --format=%an ${GIT_COMMIT}', returnStdout: true).trim()
                        C_HASH = sh (script: 'git show -s --format=%h ${GIT_COMMIT}', returnStdout: true).trim()
                    
                        def buildCause = currentBuild.getBuildCauses()[0].shortDescription
                        echo "Build cause: ${buildCause}"

                        if(params.image_ton_labs_types) {
                            G_images.put('ton-labs-types', params.image_ton_labs_types)
                        } else {
                            G_images.put('ton-labs-types', "tonlabs/ton-labs-types:source-${G_commit}")
                        }
                        env.IMAGE = G_images['ton-labs-types']
                        echo "Build cause: ${currentBuild.getBuildCauses()[0].shortDescription}"
                    }
                }
            }
        }
        stage('Versioning') {
            steps {
                script {
                    lock('bucket') {
                        withAWS(credentials: 'CI_bucket_writer', region: 'eu-central-1') {
                            identity = awsIdentity()
                            s3Download bucket: 'sdkbinaries.tonlabs.io', file: 'version.json', force: true, path: 'version.json'
                        }
                    }
                    if(params.common_version) {
                        G_binversion = sh (script: "node tonVersion.js --set ${params.common_version} .", returnStdout: true).trim()
                    } else {
                        G_binversion = sh (script: "node tonVersion.js .", returnStdout: true).trim()
                    }
                }
            }
        }
        stage('Prepare image') {
            steps {
                script {
                    docker.withRegistry('', G_docker_creds) {
                        args = "--pull --no-cache --label 'git-commit=${GIT_COMMIT}' --force-rm --target ton-labs-types-src ."
                        G_docker_image = docker.build(
                            G_images['ton-labs-types'], 
                            args
                        )
                        echo "Image ${G_docker_image} as ${G_images['ton-labs-types']}"
                        G_docker_image.push()
                    }
                }
            }
        }
        stage('Build') {
            agent {
                dockerfile {
                    registryCredentialsId "${G_docker_creds}"
                    additionalBuildArgs "--pull --target ton-labs-types-rust " + 
                                        "--build-arg \"TON_LABS_TYPES_IMAGE=${G_images['ton-labs-types']}\""
                }
            }
            steps {
                script {
                    sh """
                        cd /tonlabs/ton-labs-types
                        cargo update
                        cargo build --release
                    """
                }
            }
        }
        stage('Tests') {
            agent {
                dockerfile {
                    registryCredentialsId "${G_docker_creds}"
                    additionalBuildArgs "--pull --target ton-labs-types-rust " + 
                                        "--build-arg \"TON_LABS_TYPES_IMAGE=${G_images['ton-labs-types']}\""
                }
            }
            steps {
                script {
                    sh """
                        cd /tonlabs/ton-labs-types
                        cargo update
                        cargo test --release
                    """
                }
            }
        }
    }
    post {
        always {
            node('master') {
                script {
                    cleanWs notFailBuild: true
                }
            } 
        }
    }
}