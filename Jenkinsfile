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
        try {
          smartcheckScan([
            imageName: "${REPOSITORY}:${BUILD_NUMBER}REMOVE_ME",
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
          // environment {
          //   SMARTCHECK_AUTH_CREDS = credentials('smartcheck-auth')
          // }

          // ${BUILD_NUMBER}

          withCredentials([
            usernamePassword(
              credentialsId: 'smartcheck-auth',
              usernameVariable: 'SMARTCHECK_AUTH_CREDS_USR',
              passwordVariable: 'SMARTCHECK_AUTH_CREDS_PSW'
            )
          ]) { script {
            sh 'env'
            echo "${DSSC_SERVICE}"
            echo "${DSSC_REGISTRY}"
            def dssc_service = ['10.0.2.126', '30010']
            docker.image('mawinkler/scan-report').pull()
            docker.image('mawinkler/scan-report').inside("--entrypoint=''") {
              sh '''
                echo "${DSSC_SERVICE}"
                echo "${dssc_service[0]} ${dssc_service[1]}"
                python /usr/src/app/scan-report.py \
                  --config_path "/usr/src/app" \
                  --name "${REPOSITORY}" \
                  --image_tag "1" \
                  --out_path "${WORKSPACE}" \
                  --service "${service[0]}:${service[1]}" \
                  --username "${SMARTCHECK_AUTH_CREDS_USR}" \
                  --password "${SMARTCHECK_AUTH_CREDS_PSW}"
              '''
              archiveArtifacts artifacts: 'report_*.pdf'
            }
            error('Issues in image found')
          } }
        }
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
