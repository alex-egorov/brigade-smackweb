podTemplate(label: 'mypod', containers: [
    containerTemplate(name: 'docker', image: 'docker', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.8.0', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'helm', image: 'lachlanevenson/k8s-helm:latest', command: 'cat', ttyEnabled: true)
  ],
  volumes: [
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
  ]) {

    node('mypod') {

        checkout scm


        //GIT_BRANCH=$(git rev-parse --symbolic-full-name --abbrev-ref HEAD)
        //GIT_COMMIT=$(git rev-parse --short HEAD)
        //BUILD_DATE=$(date +"%Y-%m-%d %H-%M-%S")

        echo "Branch name: ${env.BRANCH_NAME}"
        echo "Commit ID: ${env.COMMIT_ID}"
        echo "Build Number: ${env.BUILD_NUMBER}"

        DOCKER_TAG = "${env.BRANCH_NAME}-${enc.COMMIT_ID}"
        echo "Docker Tag: $DOCKER_TAG"

        stage('Build go binaries') {
            container('golang') {

               sh """
               sh "GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o target/smackweb"
               
               archiveArtifacts artifacts: 'target/*', fingerprint: true
            }
        }

        stage('Build and push docker image') {
            container('docker') {

                //withCredentials([[$class: 'UsernamePasswordMultiBinding',
                //        credentialsId: 'dockerhub',
                //        usernameVariable: 'DOCKER_HUB_USER',
                //        passwordVariable: 'DOCKER_HUB_PASSWORD']]) {

                    dir ('.') {

                        docker.withRegistry('https://registry.example.com', 'dockerhub') {

                            def image = docker.build("alex202/smartweb:${DOCKER_TAG}")
                            image.push()
                            //image.push('latest')

                            //sh """
                            //  docker build
                            //  docker tag ubuntu ${env.DOCKER_HUB_USER}/ubuntu:${env.BUILD_NUMBER}
                            //  """
                            //sh "docker login -u ${env.DOCKER_HUB_USER} -p ${env.DOCKER_HUB_PASSWORD} "
                            //sh "docker push ${env.DOCKER_HUB_USER}/ubuntu:${env.BUILD_NUMBER} "
                        }
                    }
                }
            }
        }

        stage('do some kubectl work') {
            container('kubectl') {

                    sh "kubectl get nodes --all-namespaces"
                }
            }
        }
        stage('do some helm work') {
            container('helm') {

                dir("charts") {

                    sh "helm ls"
                    sh "helm lint smartweb"
                    sh "helm upgrade --dry-run --debug -i smartweb smartweb"
                }
            }
        }
    }
}