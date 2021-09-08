pipeline {
  agent any
  
  environment {
    scannerHome = tool 'SonarQubeScanner';
    dockerPort="7400"
    dockerRegistryUsername="gaganshera"
    username="gaganjotsingh02"
  }
  tools {
    nodejs "nodejs"
    dockerTool 'Test_Docker'
  }
  options {
    timestamps()

    timeout(time: 1, unit: 'HOURS')

    buildDiscarder(logRotator(daysToKeepStr: '10', numToKeepStr: '20'))

    parallelsAlwaysFailFast()
  }

  stages {
    stage('Build') {
      steps {
        sh 'npm i'
      }
    }
    stage('Unit Testing') {
      steps {
        sh 'npm test'
      }
    }
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('Test_Sonar') {
          sh "${scannerHome}/bin/sonar-scanner"
        }
        sleep 10
        timeout(time: 30, unit: 'SECONDS') {
            waitForQualityGate abortPipeline: true
        }
      }
    }
    stage('Docker Image') {
      steps {
        sh "docker build -t ${dockerRegistryUsername}/i-${username}-${env.BRANCH_NAME}:${BUILD_NUMBER} --no-cache ."
      }
    }
    stage('PrecontainerCheck') {
      steps {
        sh '''
          CONTAINER_ID=$(docker ps -a | grep "${dockerPort}" | cut -d " " -f 1)
          if [ $CONTAINER_ID ]
          then
              docker rm -f $CONTAINER_ID
          fi
        '''
      }
    }
    stage('PushtoDockerHub') {
      steps {
        sh "docker tag ${dockerRegistryUsername}/i-${username}-${env.BRANCH_NAME}:${BUILD_NUMBER} ${dockerRegistryUsername}/i-${username}-${env.BRANCH_NAME}:latest"
        script {
          withDockerRegistry(credentialsId: 'DockerHub', toolName: 'Test_Docker') {
            sh "docker push ${dockerRegistryUsername}/i-${username}-${env.BRANCH_NAME}:${BUILD_NUMBER}"
            sh "docker push ${dockerRegistryUsername}/i-${username}-${env.BRANCH_NAME}:latest"
          }
        }
      }
    }
    stage('Containers') {
      parallel {
        stage('Docker deployment') {
          steps {
            sh "docker run -d --name c-${username}-${env.BRANCH_NAME} -p ${dockerPort}:3010 ${dockerRegistryUsername}/i-${username}-${env.BRANCH_NAME}:latest"
          }
        }
        stage('Kubernetes Deployment') {
          steps {
            sh 'kubectl apply -f k8s/deployment.yaml'
          }
        }
      }
    }
  }
}
