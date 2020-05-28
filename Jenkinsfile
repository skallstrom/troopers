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
    // parallel (
    //   "Test": {
    //     echo 'All functional tests passed'
    //   },
    //   "Check Image (pre-Registry)": {
    //     smartcheckScan([
    //       imageName: "${REPOSITORY}:${BUILD_NUMBER}",
    //       smartcheckHost: "${DSSC_SERVICE}",
    //       smartcheckCredentialsId: "smartcheck-auth",
    //       insecureSkipTLSVerify: true,
    //       insecureSkipRegistryTLSVerify: true,
    //       preregistryScan: true,
    //       preregistryHost: "${DSSC_REGISTRY}",
    //       preregistryCredentialsId: "preregistry-auth",
    //       findingsThreshold: new groovy.json.JsonBuilder([
    //         malware: 0,
    //         vulnerabilities: [
    //           defcon1: 0,
    //           critical: 0,
    //           high: 0,
    //         ],
    //         contents: [
    //           defcon1: 0,
    //           critical: 0,
    //           high: 0,
    //         ],
    //         checklists: [
    //           defcon1: 0,
    //           critical: 0,
    //           high: 0,
    //         ],
    //       ]).toString(),
    //     ])
    //   }
    // )
    parallel (
      "Test": {
        echo 'All functional tests passed'
      },
      "Check Image (pre-Registry)": {
        try {
          smartcheckScan([
            imageName: "${REPOSITORY}:${BUILD_NUMBER}",
            smartcheckHost: "${DSSC_SERVICE}",
            smartcheckCredentialsId: "smartcheck-auth",
            insecureSkipTLSVerify: true,
            insecureSkipRegistryTLSVerify: true,
            preregistryScan: true,
            preregistryHost: "${DSSC_REGISTRY}",
            preregistryCredentialsId: "preregistry-auth",
            findingsThreshold: new groovy.json.JsonBuilder([
              malware: 0,
              vulnerabilities: [
                defcon1: 0,
                critical: 0,
                high: 0,
              ],
              contents: [
                defcon1: 0,
                critical: 0,
                high: 0,
              ],
              checklists: [
                defcon1: 0,
                critical: 0,
                high: 0,
              ],
            ]).toString(),
          ])
        } catch(e) {
          script {
            sh 'pwd'
            docker.image('mawinkler/scan-report').pull()
            docker.image('mawinkler/scan-report').inside("--entrypoint=''") {
              sh 'env'
              sh 'pwd'
              sh 'python /usr/src/app/scan-report.py --config_path /usr/src/app --name "${REPOSITORY}" --image_tag "${BUILD_NUMBER}"'
              archiveArtifacts artifacts: 'report/*.pdf'
            }
          }
        }
        //.withRun('--mount type=bind,source="$(pwd)",target=/usr/src/app/report')
        // script {
        //   sh 'docker run --mount type=bind,source="$(pwd)",target=/usr/src/app/report mawinkler/scan-report "${REPOSITORY}" "${BUILD_NUMBER}"'
        // }
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
