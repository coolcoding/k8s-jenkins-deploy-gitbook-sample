def label = "slave-${UUID.randomUUID().toString()}"

podTemplate(label: label, cloud: "kubernetes" , containers: [
  containerTemplate(name: 'docker', image: 'cnych/jenkins:jnlp6', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'maven', image: 'maven:alpine', command: 'cat', ttyEnabled: true),
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
        def dockerRegistryUrl = "dockerreg.domain"
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
            container('docker') {
              sh "sed -i 's#<BUILD_TAG>#${imageTag}#' deployment/*.yaml"
              sh "sed -i 's#<IMAGE>#${imageRepo}/tcpdump-book#' deployment/*.yaml"
              kubernetesDeploy configs: 'deployment/', kubeconfigId: 'kubectl-admin'
            } 
        }

 

    }
}
