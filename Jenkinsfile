#!/usr/bin/env groovy
import java.text.SimpleDateFormat
def version = ""
def serviceName = "adapter"
//def serviceDirectory = "adapter"
def skipBuild = false
def deployCI = false
def deployQA = false
def deployProd = false
//def scanPaths = ["adapter", "cmsdal", "oo-commons"]
def registry = ""
def ciGroup = "dev"
def qaGroup = "staging"
def prodGroup = "prod"
//def buildServiceUrl = "https://5yfganz8bb.execute-api.us-east-1.amazonaws.com/prod/service/${serviceName}/builds"

pipeline {
  agent none
  stages {
    stage("Prepare") {
      agent any
      environment {
        AWS_SECRET_KEY = credentials('AWS_SECRET_KEY')
      }
      steps {
        script {
          def dateFormat = new SimpleDateFormat("yyyyMMdd")
          def date = new Date()
          def format = dateFormat.format(date)
          def branchName = "${BRANCH_NAME}"
          def branchInfo = branchName.tokenize('-')
          if (branchName == "master") {
            deployCI = true
            version = VersionNumber (versionNumberString: '${BUILDS_TODAY}', versionPrefix: "${serviceName}-b${format}.")
            registry = "raj11646"
          } else {
            deployQA = true
            deployProd = true
            def branchBuildNumber = branchInfo.last()
            version = VersionNumber (versionNumberString: '${BUILD_NUMBER}', versionPrefix: "${serviceName}-${branchBuildNumber}.")
            registry = "rajmca10"
          }
        }
        script {
          try {
            def lastTagName = sh (
              script: "curl -H 'x-api-key: ${AWS_SECRET_KEY}' ${buildServiceUrl}/${BRANCH_NAME}",
              returnStdout: true
            )
            def lastBuildTag = lastTagName.replace("\"", "")
            scanPaths.each {
              echo "Checking for change in ${it}"
              sh "git diff --quiet --exit-code ${lastBuildTag}..HEAD ${it}"
            }
            echo "No change in ${serviceName}"
            skipBuild = true
          } catch (err) {
            echo "No previous build for this commit exists: ${err}"
            skipBuild = false
          }
        }
      }
    }

    stage('Quality Analysis') {
      when {
        expression {
          return !skipBuild
        }
      }
      parallel {
        stage ("Build") {
          agent any
          when {
            expression {
              return !skipBuild
            }
          }
          steps {
            echo "Build"
            echo "Build Number ${version}"
              sh "../gradlew buildNeeded -PprojVersion=${version} --profile -x test"
            
          }
        }

        stage ("Unit Test") {
          agent any
          steps {
            echo "Run unit tests - ${version}"
            dir(serviceDirectory) {
              sh "../gradlew test"
            }
          }
          post {
            always {
              script {
                try {
                  echo "Capture test results - ${version}"
                  junit "**/build/test-results/**/*.xml"
                } catch (err) {
                  echo "No test results found"
                }
              }
            }
          }
        }

        stage ("Integration test") {
          agent any
          steps {
            echo "Run Integration Tests"
          }
          post {
            always {
              script {
                try {
                  echo "Capture test results"
                  junit "**/build/test-results/**/*.xml"
                } catch (err) {
                  echo "No test results found"
                }
              }
            }
          }
        }
      }
    }

    stage ("Publish") {
      agent any
      environment {
        AWS_SECRET_KEY = credentials('AWS_SECRET_KEY')
      }
      when {
        expression {
          return !skipBuild
        }
      }
      steps {
        echo "Docker - build, tag & publish"
        sh "./gradlew clean"
        script {
          def branchName = "${BRANCH_NAME}"
          if (branchName == "master") {
            dir(serviceDirectory) {
              sh "../gradlew publishDocker -x test -PprojGroup=${ciGroup} -PprojVersion=${version} -PprojRegistry=${registry}"
            }
            echo "cleaning intermediate image"
            sh "docker rmi ${registry}/${ciGroup}/${serviceName}:${version}"
          } else {
            dir(serviceDirectory) {
              sh "../gradlew publishDocker -x test -PprojGroup=${qaGroup} -PprojVersion=${version} -PprojRegistry=${registry}"
            }
            echo "cleaning intermediate image"
            sh "docker rmi ${registry}/${qaGroup}/${serviceName}:${version}"
          }
        }
      }
      post {
        success {
          script {
            if (!skipBuild) {
              echo "Tagging successful build"
              sh "git tag -am 'Automated Build ${version}' $version"
              sh "git push origin $version:refs/tags/$version"
              def data = "{\"buildLabel\": \"master\", \"buildValue\": \"${version}\"}"
              sh "curl -XPATCH -H 'x-api-key: ${AWS_SECRET_KEY}' -H 'Content-type: application/json' -d '${data}' '${buildServiceUrl}'"
            } else {
              echo "Tagging skipped as build has been skipped"
            }
          }
        }
      }
    }

    // stage ("Deploy CI") {
    //   agent any
    //   when {
    //     expression {
    //       return deployCI && !skipBuild
    //     }
    //   }
    //   steps {
    //     echo "Deploy to CI - ${version}"
    //     sh 'rm -rf prana-services'
    //     sh 'git clone cicd@code.appranix.net:prana/prana-services.git'
    //     script {
    //       dir ('prana-services/k8s-yaml/adapter') {
    //         echo "Deploy using ax-cli."
    //         sh "ax -u -f ax-ci-adapter.yaml --variable-file ax-ci-variables.ini --variables imageName=${registry}/${ciGroup}/${serviceName}:${version} environment=ci envGroup=dev"
    //       }
    //     }
    //   }
    //   post {
    //     always {
    //       archiveArtifacts "prana-services/k8s-yaml/adapter/deployment.yml"
    //       archiveArtifacts "prana-services/k8s-yaml/adapter/ax-ci-variables.ini"
    //     }
    //     success {
    //       sh "docker pull ${registry}/${ciGroup}/${serviceName}:${version}"
    //       sh "docker tag ${registry}/${ciGroup}/${serviceName}:${version} ${registry}/${ciGroup}/${serviceName}:latest"
    //       sh "docker push ${registry}/${ciGroup}/${serviceName}:latest"
    //       sh "docker rmi ${registry}/${ciGroup}/${serviceName}:${version} ${registry}/${ciGroup}/${serviceName}:latest"
    //     }
    //   }
    // }

    stage ("Deploy QA") {
      agent any
      when {
        expression {
          return deployQA && !skipBuild
        }
      }
      steps {
        echo "Deploy to QA - ${version}"
        sh 'rm -rf prana-services'
        sh 'git clone cicd@code.appranix.net:prana/prana-services.git'
        script {
          dir ('prana-services/k8s-yaml/adapter') {
            echo "Deploy using ax-cli."
            sh "ax -u -f ax-qa-adapter.yaml --variable-file ax-qa-variables.ini --variables imageName=${registry}/${qaGroup}/${serviceName}:${version} environment=qa envGroup=QA"
          }
        }
      }
      post {
        always {
          archiveArtifacts "prana-services/k8s-yaml/adapter/deployment.yml"
          archiveArtifacts "prana-services/k8s-yaml/adapter/ax-qa-variables.ini"
        }
        success {
          sh "docker pull ${registry}/${qaGroup}/${serviceName}:${version}"
          sh "docker tag ${registry}/${qaGroup}/${serviceName}:${version} ${registry}/${qaGroup}/${serviceName}:latest"
          sh "docker push ${registry}/${qaGroup}/${serviceName}:latest"
          sh "docker rmi ${registry}/${qaGroup}/${serviceName}:${version} ${registry}/${qaGroup}/${serviceName}:latest"
        }
      }
    }

    stage ("Approve Deployment") {
      when {
        expression {
          return deployProd
        }
      }
      steps {
        input message: "Proceed?"
      }
    }

    stage ("Create Production Image") {
      agent any
      when {
        expression {
          return deployProd && !skipBuild
        }
      }
      steps {
        echo "Creating images for prodcution deploy"
        sh "docker pull ${registry}/${qaGroup}/${serviceName}:${version}"
        sh "docker tag ${registry}/${qaGroup}/${serviceName}:${version} ${registry}/${prodGroup}/${serviceName}:${version}"
        sh "docker push ${registry}/${prodGroup}/${serviceName}:${version}"
        sh "docker rmi ${registry}/${qaGroup}/${serviceName}:${version} ${registry}/${prodGroup}/${serviceName}:${version}"
      }
    }

    stage ("Deploy Prod") {
      agent any
      when {
        expression {
          return deployProd && !skipBuild
        }
      }
      steps {
        echo "Deploy to Production - ${version}"
        sh 'rm -rf prana-services'
        sh ''
        script {
          dir ('prana-services/k8s-yaml/adapter') {
            echo "Deploy using ax-cli."
            sh "ax -u -f ax-prod-adapter.yaml --variable-file ax-prod-variables.ini --variables imageName=${registry}/${prodGroup}/${serviceName}:${version} environment=production envGroup=Production"
          }
        }
      }
      post {
        always {
          archiveArtifacts "prana-services/k8s-yaml/adapter/deployment.yml"
          archiveArtifacts "prana-services/k8s-yaml/adapter/ax-prod-variables.ini"
        }
        success {
          sh "docker pull ${registry}/${prodGroup}/${serviceName}:${version}"
          sh "docker tag ${registry}/${prodGroup}/${serviceName}:${version} ${registry}/${prodGroup}/${serviceName}:latest"
          sh "docker push ${registry}/${prodGroup}/${serviceName}:latest"
          sh "docker rmi ${registry}/${prodGroup}/${serviceName}:${version} ${registry}/${prodGroup}/${serviceName}:latest"
        }
      }
    }
  }
}
