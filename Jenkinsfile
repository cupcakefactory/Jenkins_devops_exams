pipeline {
  environment {
    DOCKER_ID = "cupcakefactory"
    DOCKER_IMAGE_MOVIE = "${DOCKER_ID}/movie-service"
    DOCKER_IMAGE_CAST =  "${DOCKER_ID}/cast-service"
    DOCKER_TAG = "v.${BUILD_ID}.0"
    DOCKER_PASS = credentials("DOCKER_HUB_PASS")
    KUBECONFIG = credentials("config")
  }
  agent any
  stages {
    stage('Docker Build'){
      steps {
        script {
          sh '''
            docker build -t $DOCKER_IMAGE_MOVIE:$DOCKER_TAG ./movie-service
            sleep 6
            docker build -t $DOCKER_IMAGE_CAST:$DOCKER_TAG ./cast-service
            sleep 6
          '''
        }
      }
    }
    stage('Test Acceptance'){
      steps {
        script {
          sh '''
          docker run -d -p 8001:8000 --name jenkins_movie $DOCKER_IMAGE_MOVIE:$DOCKER_TAG uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
          sleep 10
          curl localhost:8001/api/v1/movies/docs
          docker run -d -p 8002:8000 --name jenkins_cast $DOCKER_IMAGE_CAST:$DOCKER_TAG uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
          sleep 10
          curl localhost:8002/api/v1/casts/docs

          docker rm -f jenkins_movie
          docker rm -f jenkins_cast
          '''
        }
      }
    }
    stage('Docker Push'){
      steps {
        script {
          sh '''
          docker login -u $DOCKER_ID -p $DOCKER_PASS
          docker push $DOCKER_IMAGE_MOVIE:$DOCKER_TAG
          docker push $DOCKER_IMAGE_CAST:$DOCKER_TAG
          '''
        }
      }
    }
    stage('Deploy-dev'){
      steps {
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          cat $KUBECONFIG > .kube/config
          cp charts/values.yaml values.yml
          sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
          helm upgrade --install app charts --values=values.yml --namespace jenkins-dev
          '''
        }
      }
    }
    stage('Deploy-qa'){
      steps {
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          cat $KUBECONFIG > .kube/config
          cp charts/values.yaml values.yml
          sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
          helm upgrade --install app charts --values=values.yml --namespace jenkins-qa
          '''
        }
      }
    }    
    stage('Deploy-staging'){
      steps {
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          cat $KUBECONFIG > .kube/config
          cp charts/values.yaml values.yml
          sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
          helm upgrade --install app charts --values=values.yml --namespace jenkins-staging
          '''
        }
      }
    }
    stage('Deploy prod'){
      when {
        branch 'master'
      }
      steps {
        timeout(time: 15, unit: "MINUTES") {
          input message: 'Do you want to deploy in production ?', ok: 'Yes'
        }
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          cat $KUBECONFIG > .kube/config
          cp charts/values.yaml values.yml
          sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
          helm upgrade --install app charts --values=values.yml --namespace jenkins-prod
          '''
        }
      }
    }
  }
  post {
    failure {
      echo "This will run if the job failed"
      mail to: "lanoireaude@gmail.com",
           subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} has failed",
           body: "For more info on the pipeline failure, check out the console output at ${env.BUILD_URL}"
    }
  }
}