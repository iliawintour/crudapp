pipeline {
  agent any
  environment {
    APP_NAME = 'app'
    CANARY_APP_NAME = 'app-canary'
    DOCKER_HUB_USER = 'iliawintour'
    BACKEND_IMAGE_NAME = 'crudback'
    DATABASE_IMAGE_NAME = 'mysql'
    MANAGER_IP = '192.168.0.1'
    BACK_DOCKERFILE = 'php.Dockerfile'
    DB_DOCKERFILE = 'mysql.Dockerfile'
  }
  stages {
    stage('Build') {
      steps {
        sh '''
          docker build -f $BACK_DOCKERFILE -t $DOCKER_HUB_USER/$BACKEND_IMAGE_NAME:$BUILD_NUMBER .
          docker build -f $DB_DOCKERFILE -t $DOCKER_HUB_USER/$DATABASE_IMAGE_NAME:$BUILD_NUMBER .
        '''
      }
    }
    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            docker push $DOCKER_HUB_USER/$BACKEND_IMAGE_NAME:$BUILD_NUMBER
            docker push $DOCKER_HUB_USER/$DATABASE_IMAGE_NAME:$BUILD_NUMBER
          '''
        }
      }
    }
    stage('Deploy Canary') {
      steps {
        sh '''
          docker stack rm $CANARY_APP_NAME || true
          sleep 10
          docker stack deploy -c docker-compose_canary.yaml $CANARY_APP_NAME
          sleep 40
        '''
      }
    }
    stage('Test Canary') {
      steps {
        sh '''
          ok=0
          for i in $(seq 1 10); do
            if curl -f -s --max-time 15 http://$MANAGER_IP:8081/ -o /tmp/canary.html && ! grep -iqE "error|fatal|exception|failed|warning" /tmp/canary.html; then
              ok=$((ok+1))
            fi
            sleep 6
          done
          echo "Результат: $ok из 10 успешных"
          [ $ok -ge 8 ]
        '''
      }
    }
    stage('Gradual Traffic Shift') {
      steps {
        sh '''
          IMG=$DOCKER_HUB_USER/$BACKEND_IMAGE_NAME:$BUILD_NUMBER
          if docker service ls --format '{{.Name}}' | grep -q "^${APP_NAME}_web-server$"; then
            docker service update --update-parallelism 1 --update-delay 10s --image $IMG ${APP_NAME}_web-server
          else
            docker stack deploy -c docker-compose.yaml $APP_NAME
          fi
        '''
      }
    }
  }
  post {
    always {
      sh 'docker stack rm $CANARY_APP_NAME || true'
    }
    failure {
      echo 'Canary не прошёл, основная версия на порту 8080 не изменилась'
    }
  }
}
