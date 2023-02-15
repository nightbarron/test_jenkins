def image = ''

pipeline {
  agent any

  stages {
    stage ('Prepare') {
      steps {
        script {
          SSH_HOST = "127.0.0.1"
          SSH_USERNAME = "root"

          def AUTHOR = ""
          def COMMIT_MSG = ""

          def changeFolders = []

          def changeLogSets = currentBuild.changeSets
          for (int i = 0; i < changeLogSets.size(); i++) {
            def entries = changeLogSets[i].items
            for (int j = 0; j < entries.length; j++) {
              def entry = entries[j]
              echo "${entry.commitId} by ${entry.author} on ${new Date(entry.timestamp)}: ${entry.msg}"
              
              AUTHOR = entry.author
              COMMIT_MSG += " %0A - ${entry.msg} "
            }
          }

          if ("${env.BRANCH_NAME}" == "master") {
            env.APP_NAME = "app.vietnix.vn"
            env.APP_ENV = "prod"
            env.DEPLOYMENT_NAME = "vnx-app-prod"
            env.IMAGE_NAME = "vietnix/app"
          } else if ("${env.BRANCH_NAME}" == "dev") {
            env.APP_NAME = "appdev.vietnix.vn"
            env.APP_ENV = "dev"
            env.DEPLOYMENT_NAME = "vnx-app-dev"
            env.IMAGE_NAME = "vietnix/appdev"
          } else {
            error "Branch is invalid!"
          }

          env.AUTHOR = AUTHOR
          env.COMMIT_MSG = COMMIT_MSG
        }
      }
    }

    stage('Build') {
      steps {
        script {
          image = docker.build("${env.IMAGE_NAME}", "-f Dockerfile.${env.APP_ENV} .")
        }
      }
    }

    stage('Push') {
      steps {
        script {
          docker.withRegistry('https://registry.k8s.vietnix.xyz', 'registry_k8s_vietnix_xyz') {
            image.push()
          }
        }
      }
    }

    stage ('Deploy') {
     steps {
        script {
          withKubeConfig([credentialsId: 'kubeconfig', serverUrl: 'https://kubernetes.default.svc.cluster.local' ]) {
            sh """kubectl apply -f deployment.${env.APP_ENV}.yaml"""
            sh """kubectl patch deployment ${env.DEPLOYMENT_NAME} -p '{ \"spec\":{\"template\":{\"metadata\":{\"labels\":{\"buildNumber\":\"$BUILD_NUMBER\"}}}}}' """
          }
        }
      }
    }
  }

  post {
    always {
      cleanWs()
    }

    success {
      sh """
        curl -s 'https://api.telegram.org/bot368982987:AAFgZsBlQxqJ8R9P6THC7m16zApIWKvqQx8/sendMessage?chat_id=-548238904&parse_mode=markdown&text=*SUCCESS* %0A %0A *${env.APP_NAME}* %0A %0A Branch: ${env.BRANCH_NAME} %0A %0A Author: ${env.AUTHOR}' > /dev/null
      """
    }

    failure {
      sh """
        curl -s 'https://api.telegram.org/bot368982987:AAFgZsBlQxqJ8R9P6THC7m16zApIWKvqQx8/sendMessage?chat_id=-548238904&parse_mode=markdown&text=*FAILED* %0A %0A *${env.APP_NAME}* %0A %0A Branch: ${env.BRANCH_NAME} %0A %0A Author: ${env.AUTHOR}' > /dev/null
      """
    }
  }
}
