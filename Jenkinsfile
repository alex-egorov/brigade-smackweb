#!/usr/bin/env groovy

PROJECT_NAME = 'brigade-smackweb'

podTemplate(label: 'mypod', containers: [
    containerTemplate(name: 'golang', image: 'golang:1.8', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'docker', image: 'docker', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.8.0', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'helm', image: 'lachlanevenson/k8s-helm:latest', command: 'cat', ttyEnabled: true)
  ],
  envVars: [
          envVar(key: 'BRANCH_NAME', value: env.BRANCH_NAME),
          envVar(key: 'COMMIT_ID', value: env.COMMIT_ID),
          envVar(key: 'BUILD_NUMBER', value: env.BUILD_NUMBER)
  ],
  volumes: [
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
  ]) {

    node('mypod') {

        checkout scm

        // print environment variables
        echo sh(script: 'env|sort', returnStdout: true)

        gitCommit = sh returnStdout: true, script: 'git rev-parse HEAD'
        gitCommit = gitCommit.trim()
        // git branch name is taken from an env var for multi-branch pipeline project, or from git for other projects
        if (env['BRANCH_NAME']) {
            gitBranch = BRANCH_NAME
        } else {
            gitBranch = sh returnStdout: true, script: 'git rev-parse --abbrev-ref HEAD'
        }
        gitBranch = gitBranch.trim()

        echo "Branch name: $"
        echo "Commit ID: ${env.COMMIT_ID}"
        echo "Build Number: ${env.BUILD_NUMBER}"

        imageTag = "${gitBranch}-${gitCommit}"
        def buildInfo = """# Build info
BUILD_NUMBER=${env.BUILD_NUMBER}
BUILD_GIT_COMMIT=${gitCommit}
BUILD_GIT_BRANCH=${gitBranch}
DOCKER_IMAGE_TAG=${imageTag}
"""

        echo $buildInfo

        stage('Build go binaries') {
            container('golang') {

                def pwd = pwd()

                sh """
                    mkdir -p /go/src/github.com/alex-egorov
                    ln -s $pwd /go/src/github.com/alex-egorov/$PROJECT_NAME
                    cd /go/src/github.com/alex-egorov/$PROJECT_NAME
                    go get && GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o target/smackweb
                """
            }
        }

        stage('Build and push docker image') {
            container('docker') {

                withCredentials([[$class: 'UsernamePasswordMultiBinding',
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_HUB_USER',
                        passwordVariable: 'DOCKER_HUB_PASSWORD']]) {

                    sh """
                      docker build --force-rm \
                            --build-arg IMAGE_TAG_REF=${imageTag} \
                            --build-arg VCS_REF=${gitCommit} \
                            -t ${DOCKER_HUB_USER}/${PROJECT_NAME}:${imageTag} .
                      """
                    sh "docker login -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD} "
                    sh "docker push ${DOCKER_HUB_USER}/${PROJECT_NAME}:${imageTag} "
                }
            }
        }

        stage('do some kubectl work') {
            container('kubectl') {

                sh "kubectl get nodes --all-namespaces"
            }
        }
        stage('do some helm work') {
            container('helm') {

                dir("charts") {

                    sh "helm ls"

                    sh "helm lint smackweb"
                    sh "helm upgrade -i smackweb smackweb"
                }
            }
        }
    }
}