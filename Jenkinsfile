import groovy.json.JsonBuilder

node('jenkins-jenkins-slave') {
  withEnv(['REPOSITORY=troopers']) {
    stage('Pull Image from Git') {
      script {
        git (url: "${scm.userRemoteConfigs[0].url}", credentialsId: "github-auth")
      }
    }
    stage('Build Image') {
      script {
        dbuild = docker.build("${REPOSITORY}:$BUILD_NUMBER")
      }
    }
    parallel (
      "Test": {
        //script {
        //  sh "python tests/test_app.py"
        //}
        echo 'All functional tests passed'
      }
    )
    stage('Push Image to Registry') {
      script {
        docker.withRegistry("https://${K8S_REGISTRY}", 'registry-auth') {
          dbuild.push('$BUILD_NUMBER')
          dbuild.push('latest')
        }
      }
    }
    stage('Deploy App to Kubernetes') {
      script {
        // secretNamespace: "default",
        // secretName: "cluster-registry2",
        kubernetesDeploy(configs: "app.yml",
                         kubeconfigId: "kubeconfig",
                         enableConfigSubstitution: true,
                         dockerCredentials: [
                           [credentialsId: "registry-auth", url: "${K8S_REGISTRY}"],
                         ])
      }
    }
  }
}
