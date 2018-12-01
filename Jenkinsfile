pipeline {
    agent {
        kubernetes {
            label "builder-pod-${UUID.randomUUID().toString()}"
            defaultContainer 'jnlp'
            serviceAccount "build"
            yamlFile "CiKubernetesPod.yaml"
        }
    }

    environment {
        project="go-demo-4"
        image="vfarcic/go-demo-4"
        domain = "acme.com"
        cmAddr = "cm.acme.com"

        rsaKey="go-demo-rsa-key"
        githubToken="GITHUB_TOKEN"
    }


    options {
        buildDiscarder logRotator(numToKeepStr: '5')
        disableConcurrentBuilds()
    }

    stages {
        stage('build') {
            steps {
                ciPrettyBuildNumber()

                container('git') {
                    ciWithGitKey(params.rsaKey) {                        
                        // publish env vars
                        ciBuildEnvVars() 
                    }
                }

                container('docker') {
                    ciK8sBuildImage(params.image, false, env.BUILD_TAG)
                }
            }
        }

        stage("func-test") {
            steps {
                container("helm") {
                    ciK8sUpgradeBeta(params.project, params.domain, env.BUILD_TAG)
                }

                container("kubectl") {
                    ciK8sRolloutBeta(params.project)
                }

                container("golang") {
                    ciK8sFuncTestGolang(params.project, params.domain)
                }
            }

            post {
                failure {
                    container("helm") {
                        ciK8sDeleteBeta(params.project)
                    }
                }
            }
        }


        stage("Release") {
            when {
                anyOf {
                    branch "master"
                    branch "hotfix-*"
                }
            }

            steps {
                script {
                    container("git") {
                        ciWithGitKey(params.rsaKey) {
                            env.RELEASE_TAG = ciSuggestVersion(ciVersionRead())
                        }
                    }

                    container("helm") {
                        // you can have your master builds running for some time until someone decides to promote it.
                        // better to autodelete the deployment when it times out
                        
                        ciContinueAfterTimeout(5, 'MINUTES') {
                            ciConditionalInputExecution(
                                    id: "Release Gate",
                                    message: "Release ${params.project} ?",
                                    ok: "ok",
                                    name: "release") {
                                //docker tag
                                container('docker') {
                                    ciRetag(env.BUILD_TAG, false, ["latest", env.shortGitCommit, env.RELEASE_TAG])
                                }

                                //git tag
                                container('git') {
                                    ciWithGitKey(params.rsaKey) {
                                        ciTagGitRelease(tag: env.RELEASE_TAG)
                                    }
                                }

                                container('gren') {
                                    //https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/
                                    withCredentials([string(credentialsId: params.githubToken, variable: 'TOKEN')]) {
                                        sh "gren release --token=${TOKEN}"
                                    }
                                }

                                container("helm") {
                                    ciK8sPushHelm(params.project, env.RELEASE_TAG, params.cmAddr, true)
                                }
                            }
                        }
                    
                    }

                }
            }
        }
    }

    post {
        always {
            ciWhenNotReleaseBranches {
                container("helm") {
                        ciK8sDeleteBeta(params.project)                        
                }
            }
        }
    }
}