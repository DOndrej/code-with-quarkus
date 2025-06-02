@Library("SPUtil") _

import hudson.model.*
import jenkins.model.*
import hudson.tasks.test.AbstractTestResultAction


def branchFolder = env.BRANCH_NAME.replaceAll('/', '_')
def WORKSPACE = "/home/jenkins/agent/branch-ws/"+branchFolder+"/ai_translate"

def dockerReleaseHost = 'docker-releases.kubernetes-kombit.eu-de.containers.appdomain.cloud'
def dockerSnapshotHost = 'docker-snapshots.kubernetes-kombit..eu-de.containers.appdomain.cloud'

def base_images_repo = ""

if(params.CREATE_RELEASE) {
    base_images_repo = dockerReleaseHost
} else {
    base_images_repo = dockerSnapshotHost
}


pipeline {
    agent any
    tools {
        maven 'maven 3.6.1'
    }


    environment {
        MVN_LOCAL_REPO = "/home/jenkins/agent/branch-ws/${branchFolder}/.m2"
        NAMESPACE = "ai-translate-app"
    }
    options {
        buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10'))
        skipDefaultCheckout()
        disableConcurrentBuilds()
    }

    parameters {
        booleanParam(defaultValue: false, description: 'Whether or not to create a release', name: 'CREATE_RELEASE')
        string(name: 'VERSION_NAME', description: 'Version to use for build')
    }

    stages {

        stage('Checkout') {
            steps {
                ws ("$WORKSPACE") {
                    checkout scm
                }
            }
        }

        stage ('Determine version') {
            steps {
                ws ("$WORKSPACE") {

                    script {
                        echo "Using base_images_repo = $base_images_repo"
                    }
                   
                    
                    withDocker() {
                        script {
                            if(params.CREATE_RELEASE) {
                                def tagExists = dockerCheckExists image: "ai_translate", tag: "${params.VERSION_NAME}"
                                echo "Tag already exists: $tagExists"

                                if(tagExists) {
                                    error("Docker images already exists with tag $params.VERSION_NAME")
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Build ai-translate-app') {
            steps {
                ws ("$WORKSPACE") {
                    withConfig() {
                        sh """
                            mvn -s ${MAVEN_SETTINGS_XML} clean -B install -Dmaven.repo.local=${MVN_LOCAL_REPO} -Dproject.release.version=${params.VERSION_NAME}
                        """
                    }
                }
            }
        }


        stage('Build, tag and push ai_translate image') {
            steps {
                ws ("$WORKSPACE") {
                    script {
                        withDocker {
                            def image = docker.build(${base_images_repo}/ai_translate/ai-translate-app + ":${params.VERSION_NAME}")
                            image.push()
                            image.push("${params.VERSION_NAME}-$BUILD_NUMBER")
                            if (params.CREATE_RELEASE) {
                              image.push("latest")
                            }
                        }
                        
                    }
                }   
            }
        }
    

        stage("Deploy to Kubernetes") {
            steps {
                ws ("$WORKSPACE") {
                    withConfig() {
                        script {

                            withCredentials([file(credentialsId: 'regcred', variable: 'regcredFile')]) {
                                // Creates namespace. Ignores errors in case namespace already exists.
                                sh("set +e && kubectl create namespace ${NAMESPACE} ;set -e;echo 0 ")
                                // Add the regrisrtry credential to the namespace
                                sh "kubectl apply -n ${_NAMESPACE} -f $regcredFile"
                            }
                            // Uninstall any chart that is currently deployed to the given namespace
                            sh("helm list -a  -n ${NAMESPACE} |grep ai- |grep -o -E \"^\\S+\" |xargs -r -L1 helm uninstall -n ${NAMESPACE}")

                            // Deploy db containers
                            sh("helm upgrade --install --wait -n ${NAMESPACE} ai-translate-app ./helm/ai-translate-app  --set image.repository=${base_images_repo} --set image.tag=${params.VERSION_NAME}")
                        }
                    }
                }
            }
        }




}
}
