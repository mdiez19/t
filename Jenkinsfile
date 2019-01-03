pipeline {
  agent {
    label "jenkins-maven-serverless"
  }
  environment {
    ORG = 'mmapik8ssvc'
    APP_NAME = 't'
    CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')

  }
  stages {
    stage('PR Step') {
      when {
        anyOf {
          branch 'PR-*'
        }
      }
      steps {
        slack.configAndSendMessage(slackMessage: "Pull Request Build started, To veiw this PR, go to <$CHANGE_URL|HERE>\nAPP: ${env.APP_NAME}\nBranch: ${env.BRANCH_NAME}", slackColor: "")
        sh "export VERSION=$PREVIEW_VERSION && skaffold build -f skaffold.yaml"
        sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"

        // Generate the configmap in the config directory
        sh "kubectl create configmap ChangeMe --dry-run=true --from-file ./config -oyaml > charts/$APP_NAME/templates/settings-config.yaml"
        sh "sed  -i 's/ChangeMe/{{ template \"fullname\" . }}/' charts/$APP_NAME/templates/settings-config.yaml"
        sh "helm template charts/$APP_NAME"
        dir('charts/preview') {
          sh "make preview"
          sh "jx preview --app $APP_NAME --dir ../.."
        }
      }
    }
    stage('Build') {
      when {
        anyOf {
          branch 'feature/*'
          branch 'PR-*'
          branch 'develop'
          branch 'release'
          branch 'master'
        }
      }

      steps {
        container('maven') {
          sh "mvn -v"
          sh "mvn package"
        }
      }
    }
    stage('Build Release') {
      when {
        anyOf {
          branch 'develop'
          branch 'release'
          branch 'master'
        }
      }
      steps {
        slack.configAndSendMessage(slackMessage: "Release Has started\nAPP: ${env.APP_NAME}\nBranch: ${env.BRANCH_NAME}", slackColor: "")

        // ensure we're not on a detached head
        sh "git checkout master"
        sh "git config --global credential.helper store"
        sh "jx step git credentials"

        // so we can retrieve the version in later steps
        sh "echo \$(jx-release-version) > VERSION"
        sh "jx step tag --version \$(cat VERSION)"
        sh "export VERSION=`cat VERSION` && skaffold build -f skaffold.yaml"
        sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)"
        container('nodejs') {
          sh "echo "node: $(node -v)""
          sh "echo "npm: $(npm -v)""
          sh "npm install"
          sh "serverless -v"
          sh "serverless deploy -v --stage $BRANCH_NAME"
        }
      }
    }
    stage('Promote to Environments') {
      when {
        branch 'master'
      }
      steps {
        dir('charts/t') {
          sh "jx step changelog --version v\$(cat ../../VERSION)"

          // Generate the configmap in the config directory
          sh "kubectl create configmap ChangeMe --dry-run=true --from-file ./config -oyaml > charts/$APP_NAME/templates/settings-config.yaml"
          sh "sed  -i 's/ChangeMe/{{ template \"fullname\" . }}/' charts/$APP_NAME/templates/settings-config.yaml"
          sh "helm template charts/$APP_NAME"

          // release the helm chart
          sh "jx step helm release"

          // promote through all 'Auto' promotion Environments
          sh "jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)"
        }
      }
    }
  }
  post {
        always {
          cleanWs()
          script {
            if(currentBuild.result=="FAILURE") {
              slack.configAndSendMessage(slackMessage: "Jenkins job failed\nAPP: ${env.APP_NAME}\nBranch: ${env.BRANCH_NAME}", slackColor: "danger")
            }
            else {
              slack.configAndSendMessage(slackMessage: "Jenkins Job has passed\nAPP: ${env.APP_NAME}\nBranch: ${env.BRANCH_NAME}", slackColor: "good")
            }
          }
        }
  }
}
