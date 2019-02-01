#!/usr/bin/env groovy
import java.text.SimpleDateFormat
def version = ""
def serviceName = "spring"
def groupName = "spring"
def skipBuild = false
def ciGroup = "dev"
def qaGroup = "staging"
def scanPaths = ["circuit-ansible-1"]
def devRegistry = "raj11646"
def prodRegistry = "rajmca10"
def buildServiceUrl = "https://5yfganz8bb.execute-api.us-east-1.amazonaws.com/prod/service/admin-cli/builds"

pipeline {
  agent none
  stages {
    stage('Build & Publish') {
      agent any
      environment {
        AWS_SECRET_KEY = credentials('AWS_SECRET_KEY')
      }
      steps {
        git(url: )
        script {
          def dateFormat = new SimpleDateFormat("yyyyMMdd")
          def date = new Date()
          def format = dateFormat.format(date)
          version = VersionNumber (versionNumberString: '${BUILDS_TODAY}', versionPrefix: "${serviceName}-b${format}.")
          adminVersion = "${ADMIN_CLI_VERSION}"

          if (adminVersion == "latest") {
            def lastTagName = sh (
              script: "curl -H 'x-api-key: ${AWS_SECRET_KEY}' ${buildServiceUrl}/master",
              returnStdout: true
            )
            def lastBuildTag = lastTagName.replace("\"", "")
            if (lastBuildTag != null) {
              adminVersion = lastBuildTag
            }
          }
        }
        echo "Docker - build, tag & publish"
        withEnv(["GO_DEPENDENCY_LABEL_DIRECTOR_BASE=${adminVersion}"]) {
          sh "dockerize -template Dockerfile.tmpl:Dockerfile"
          sh "docker build -t ${devRegistry}/${ciGroup}/${serviceName}:${version} -t ${prodRegistry}/${qaGroup}/${serviceName}:${version} ."
          sh "docker push ${devRegistry}/${ciGroup}/${serviceName}:${version}"
          sh "docker push ${prodRegistry}/${qaGroup}/${serviceName}:${version}"
          sh "docker rmi ${devRegistry}/${ciGroup}/${serviceName}:${version} ${prodRegistry}/${qaGroup}/${serviceName}:${version}"
        }
      }
    }

    stage ("Approve Deployment to QA") {
        steps {
            input message: "Proceed?"
        }
    }

    stage ("Deploy QA") {
        agent any
        steps {
            echo "Deploy to QA - ${version}"
            sh 'rm -rf prana-services'
            sh 'git clone cicd@code.appranix.net:prana/prana-services.git'
            script {
                dir ('prana-services/k8s-yaml/private-director/ansible-director') {
                    echo "Deploy using ax-cli."
                    sh "ax -u -f ax-qa-director.yaml --variable-file ax-qa-variables.ini --variables imageName=${prodRegistry}/${qaGroup}/${serviceName}:${version} environment=qa envGroup=QA"
                }
            }
        }
        post {
            always {
                archiveArtifacts "prana-services/k8s-yaml/private-director/ansible-director/deployment.yml"
                archiveArtifacts "prana-services/k8s-yaml/private-director/ansible-director/ax-qa-variables.ini"
            }
            success {
                sh "docker pull ${prodRegistry}/${qaGroup}/${serviceName}:${version}"
                sh "docker tag ${prodRegistry}/${qaGroup}/${serviceName}:${version} ${prodRegistry}/${qaGroup}/${serviceName}:latest"
                sh "docker push ${prodRegistry}/${qaGroup}/${serviceName}:latest"
                sh "docker rmi ${prodRegistry}/${qaGroup}/${serviceName}:${version} ${prodRegistry}/${qaGroup}/${serviceName}:latest"
            }
        }
    }

  }
}
