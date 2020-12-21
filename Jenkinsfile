def label = "slave-${UUID.randomUUID().toString()}"


def helmDeploy(Map args) {
    sh """
      helm repo add helmchart https://reg.domain.com/chartrepo/helmchart --username=${args.username} --password=${args.password} --ca-file=chart/harbor_ca.crt
      helm repo update
    """

    if (args.dry_run) {
        println "Debug.."
        sh "helm upgrade --dry-run --debug --install ${args.name} ${args.chartDir} -f chart/values.yaml --set image.repository=${args.image} --set image.tag=${args.tag} --namespace=${args.namespace}"
    } else {
        println "deploy.."
        sh "helm upgrade --install ${args.name} ${args.chartDir} -f chart/values.yaml --set image.repository=${args.image} --set image.tag=${args.tag} --namespace=${args.namespace}"
    }
}


podTemplate(label: label, cloud: "kubernetes" , containers: [
  containerTemplate(name: 'docker', image: 'cnych/jenkins:jnlp6', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'maven', image: 'maven:alpine', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'kubectl-helm-docker', image: 'reg.domain.com/imagerepo/kubectl-helm-docker', command: 'cat', ttyEnabled: true),
], volumes: [
  secretVolume(mountPath: '/root/.kube', secretName: 'kubeconfig'),
  hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
  persistentVolumeClaim(mountPath: '/root/.m2', claimName: 'maven-pvc')
]) {
    node(label) {
        def myRepo = checkout scm
        def gitCommit = myRepo.GIT_COMMIT
        def gitBranch = myRepo.GIT_BRANCH
        def imageTag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        def dockerRegistryUrl = "xmreg.domain.com"
        def imageRepo = "${dockerRegistryUrl}/app"

        stage('docker build') {
            container('docker') {
              dir('gitbook'){
               sh """
                 docker build -t ${imageRepo}/tcpdump-book:${imageTag} .
               """
              }
           }
        }

        stage('Docker Push') {
            withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {
                container('docker') {
                  sh """
                     docker login ${dockerRegistryUrl} -u ${dockerHubUser} -p ${dockerHubPassword}
                     docker push ${imageRepo}/tcpdump-book:${imageTag}
                  """
                }
            }
        }
        stage('Deploy') {
          withCredentials([[$class: 'UsernamePasswordMultiBinding',
            credentialsId: 'dockerHub',
            usernameVariable: 'DOCKER_HUB_USER',
            passwordVariable: 'DOCKER_HUB_PASSWORD']]) {
              container('kubectl-helm-docker') {
                helmDeploy(
                    dry_run     : false,
                    name        : "tcpdump-book",
                    chartDir    : "helmchart/tcpdump-book",
                    namespace   : "testhelm",
                    tag         : "${imageTag}",
                    image       : "${imageRepo}/tcpdump-book",
                    username    : "${DOCKER_HUB_USER}",
                    password    : "${DOCKER_HUB_PASSWORD}"
                )
                echo "Helm deploy success..."
              }
          }
        }
    }
}
