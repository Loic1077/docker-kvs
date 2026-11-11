pipeline {
  agent any
  environment {
    DOCKERHUB_USER = "oca2404pg0004"
    BUILD_HOST = "root@172.22.27.238"
    PROD_HOST = "root@172.22.27.239"
    BUILD_TIMESTAMP = sh(script: "date +%Y%m%d-%H%M%S", returnStdout: true).trim()
  }
  stages {
    stage('Pre Check') {
      steps {
        sh "test -f ~/.docker/config.json"
        sh "cat ~/.docker/config.json | grep docker.io"
      }
    }
    stage('Build') {
      steps {
        // Zeigt im Log, welche Dateien Jenkins wirklich baut
        sh "cat docker-compose.build.yml"
        sh "cat app/Dockerfile"
        sh "docker-compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml down"
        sh "docker -H ssh://${BUILD_HOST} volume prune -f"
        // --no-cache: keine alten Schichten wiederverwenden
        sh "docker-compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml build --no-cache"
        sh "docker-compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml up -d"
        sh "docker-compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml ps -a"
      }
    }
    stage('Wait for App') {
      steps {
        // Wartet bis zu 60 Sekunden, bis die App antwortet
        sh '''
          for i in $(seq 1 30); do
            docker -H ssh://$BUILD_HOST container exec dockerkvs_apptest python -c "import requests; requests.get('http://app:80/api/v1/keys/')" && exit 0
            sleep 2
          done
          echo "App antwortet nicht"
          exit 1
        '''
      }
    }
    stage('Test') {
      steps {
        sh "docker -H ssh://${BUILD_HOST} container exec dockerkvs_apptest pytest -v test_app.py"
        sh "docker -H ssh://${BUILD_HOST} container exec dockerkvs_webtest pytest -v test_static.py"
        sh "docker -H ssh://${BUILD_HOST} container exec dockerkvs_webtest pytest -v test_selenium.py"
      }
    }
    stage('Register') {
      steps {
        sh "docker -H ssh://${BUILD_HOST} tag dockerkvs_web ${DOCKERHUB_USER}/dockerkvs_web:${BUILD_TIMESTAMP}"
        sh "docker -H ssh://${BUILD_HOST} tag dockerkvs_app ${DOCKERHUB_USER}/dockerkvs_app:${BUILD_TIMESTAMP}"
        sh "docker -H ssh://${BUILD_HOST} push ${DOCKERHUB_USER}/dockerkvs_web:${BUILD_TIMESTAMP}"
        sh "docker -H ssh://${BUILD_HOST} push ${DOCKERHUB_USER}/dockerkvs_app:${BUILD_TIMESTAMP}"
      }
    }
    stage('Deploy') {
      steps {
        sh "cat docker-compose.prod.yml"
        sh "echo 'DOCKERHUB_USER=${DOCKERHUB_USER}' > .env"
        sh "echo 'BUILD_TIMESTAMP=${BUILD_TIMESTAMP}' >> .env"
        sh "cat .env"
        sh "docker-compose -H ssh://${PROD_HOST} -f docker-compose.prod.yml up -d"
        sh "docker-compose -H ssh://${PROD_HOST} -f docker-compose.prod.yml ps"
      }
    }
  }
  post {
    always {
      // Logs ausgeben, damit Abstuerze im Console Output sichtbar sind, dann aufraeumen
      sh "docker -H ssh://${BUILD_HOST} ps -a || true"
      sh "docker -H ssh://${BUILD_HOST} logs --tail 50 dockerkvs_app || true"
      sh "docker -H ssh://${BUILD_HOST} logs --tail 50 dockerkvs_web || true"
      sh "docker-compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml down || true"
    }
  }
}