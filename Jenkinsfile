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