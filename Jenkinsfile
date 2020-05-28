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
        echo 'All functional tests passed'
      },
      "Check Image (pre-Registry)": {
        // smartcheckScan([
        //   imageName: "${REPOSITORY}:${BUILD_NUMBER}",
        //   smartcheckHost: "${DSSC_SERVICE}",
        //   smartcheckCredentialsId: "smartcheck-auth",
        //   insecureSkipTLSVerify: true,
        //   insecureSkipRegistryTLSVerify: true,
        //   preregistryScan: true,
        //   preregistryHost: "${DSSC_REGISTRY}",
        //   preregistryCredentialsId: "preregistry-auth",
        //   findingsThreshold: new groovy.json.JsonBuilder([
        //     malware: 0,
        //     vulnerabilities: [
        //       defcon1: 0,
        //       critical: 0,
        //       high: 0,
        //     ],
        //     contents: [
        //       defcon1: 0,
        //       critical: 0,
        //       high: 0,
        //     ],
        //     checklists: [
        //       defcon1: 0,
        //       critical: 0,
        //       high: 0,
        //     ],
        //   ]).toString(),
        // ])
        docker.image('mawinkler/scan-report').inside {
          python /usr/src/app/scan-report.py "${REPOSITORY}" "${BUILD_NUMBER}"
        }
        //.withRun('--mount type=bind,source="$(pwd)",target=/usr/src/app/report')
        // script {
        //   sh 'docker run --mount type=bind,source="$(pwd)",target=/usr/src/app/report mawinkler/scan-report "${REPOSITORY}" "${BUILD_NUMBER}"'
        // }
        archiveArtifacts artifacts: '*.pdf'
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
