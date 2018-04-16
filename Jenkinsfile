 imageName = 'cd-demo'
 docker_registry = 'automationteamdev.azurecr.io'
 credential_id = '8d4c4c0d-04c2-4bee-8f78-9c67e1c8b402'


 def cleanUp()
 {
    sh "docker rm -f ${imageName} || true"
    sh "docker ps -aq | xargs docker rm || true"
    sh "docker images -aq -f dangling=true | xargs docker rmi || true"
 }



 

  node("swarm-dev") {
    checkout scm

    stage("Unit Test") {
      sh "docker run --rm -v ${WORKSPACE}:/go/src/${imageName} golang go test ${imageName} -v --run Unit"
    }
    stage("Integration Test") {
      try {
        // create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
        sh "docker build -t ${imageName} ."
        // rm = remove container =f force the removal of a running container 
        sh "docker rm -f ${imageName} || true"
        // -d = create container in background
        sh "docker run -d -p 8080:8080 --name=${imageName} ${imageName}"
        // env variable is used to set the server where go test will connect to run the test
        sh "docker run --rm -v ${WORKSPACE}:/go/src/${imageName} --link=${imageName} -e SERVER=${imageName} golang go test ${imageName} -v --run Integration"
      }
      catch(e) {
        error "Integration Test failed"
      }finally {
       cleanUp()
      }
    }
    stage("Build") {
      sh "docker build -t ${imageName}:${BUILD_NUMBER} ."
      sh "docker tag ${imageName}:${BUILD_NUMBER} ${docker_registry}/${imageName}:${BUILD_NUMBER} "
    }
    stage("Publish") {
     docker.withRegistry("https://automationteamdev.azurecr.io", '8d4c4c0d-04c2-4bee-8f78-9c67e1c8b402') {
        sh "docker push ${docker_registry}/${imageName}:${BUILD_NUMBER}"
      }
    }
  }

  node("swarm-dev") {
    checkout scm

    stage("Staging") {
      try {
        sh "docker rm -f ${imageName} || true"
        docker.withRegistry("https://${docker_registry}", ${credential_id}) {
          sh "docker run -d -p 8080:8080 --name=${imageName} ${docker_registry}/${imageName}:${BUILD_NUMBER}"
          sh "docker run --rm -v ${WORKSPACE}:/go/src/${imageName} --link=${imageName} -e SERVER=${imageName} golang go test ${imageName} -v"
        }
      } catch(e) {
        error "Staging failed"
      } finally {
        cleanUp()
      }
    }
  }

  node("swarm-dev") {
    stage("Production") {
      try {
        // Create the service if it doesn't exist otherwise just update the image
        sh '''
          SERVICES=$(docker service ls --filter name=${imageName} --quiet | wc -l)
          if [[ "$SERVICES" -eq 0 ]]; then
            docker service create --replicas 3 --network swarm_overlay --name ${imageName} -p 8080:8080 ${docker_registry}/${imageName}:${BUILD_NUMBER}
          else
            docker service update --image ${docker_registry}/${imageName}:${BUILD_NUMBER} ${imageName}
          fi
          '''
        // run some final tests in production
        checkout scm
        sh '''
          sleep 60s 
          for i in `seq 1 20`;
          do
            STATUS=$(docker service inspect --format '{{ .UpdateStatus.State }}' ${imageName})
            if [[ "$STATUS" != "updating" ]]; then
              docker run --rm -v ${WORKSPACE}:/go/src/${imageName} --network swarm_overlay -e SERVER=cd-demo golang go test ${imageName} -v --run Integration
              break
            fi
            sleep 10s
          done
          
        '''
      }catch(e) {
        sh "docker service update --rollback ${imageName}"
        error "Service update failed in production"
      }finally {
        sh "docker ps -aq | xargs docker rm || true"
      }
    }
  }
