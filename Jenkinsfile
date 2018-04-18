def IMAGE_NAME = 'cd-demo'
def AZURE_REGISTRY = 'automationteamdev.azurecr.io'
def AZURE_CREDID = 'dcc9154c-828d-461d-9443-47a85bd38aae'
def CONTAINER_TAG= 'latest'

  node("swarm-qa") {
    checkout scm

    stage("Unit Test") {
      // --rm = removes container after unit test and peforms clean up | -v cleans volume
      sh "docker run --rm -v ${WORKSPACE}:/go/src/${IMAGE_NAME} golang go test ${IMAGE_NAME} -v --run Unit"
    }
    stage("Integration Test") {
      try {
        // create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
        sh "docker build -t ${IMAGE_NAME} ."
        // rm = remove container =f force the removal of a running container 
        sh "docker rm -f ${IMAGE_NAME} || true"
        // -d = create container in background
        sh "docker run -d -p 8080:8080 --name=${IMAGE_NAME} ${IMAGE_NAME}"
        // env variable is used to set the server where go test will connect to run the intergration test
        sh "docker run --rm -v ${WORKSPACE}:/go/src/${IMAGE_NAME} --link=${IMAGE_NAME} -e SERVER=${IMAGE_NAME} golang go test ${IMAGE_NAME} -v --run Integration"
      }
      catch(e) {
        error "Integration Test failed"
      }finally {
        sh "docker rm -f ${IMAGE_NAME} || true"
      }
    }
    stage("Build") {
      sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
      sh "docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${AZURE_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER} "
    }
    stage("Publish") {
     docker.withRegistry("https://${AZURE_REGISTRY}", "${AZURE_CREDID}") {
        sh "docker push ${AZURE_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}"
      }
    }
  }

  node("swarm-stage") {
    checkout scm

    stage("Staging") {
      try {
        sh "docker rm -f ${IMAGE_NAME} || true"
        docker.withRegistry("https://automationteamdev.azurecr.io", 'dcc9154c-828d-461d-9443-47a85bd38aae') {
          sh "docker run -d -p 8080:8080 --name=${IMAGE_NAME} ${AZURE_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}"
          sh "docker run --rm -v ${WORKSPACE}:/go/src/${IMAGE_NAME} --link=${IMAGE_NAME} -e SERVER=${IMAGE_NAME} golang go test ${IMAGE_NAME} -v"
        }
      } catch(e) {
        error "Staging failed"
      } finally {
        sh "docker rm -f ${IMAGE_NAME} || true"
      }
    }
  }

  node("swarm-prod") {
    stage("Production") {
      try {
        // Create the service if it doesn't exist otherwise just update the image
        sh '''
          SERVICES=$(docker service ls --filter name=${IMAGE_NAME} --quiet | wc -l)
          if [[ "$SERVICES" -eq 0 ]]; then
            docker service create --with-registry-auth \
            --name ${IMAGE_NAME} \
            --publish 8080:8080 \
            --network swarm_overlay \
            --constraint "node.labels.environment == prod" \
            --constraint "node.labels.type == agent" \
            ${AZURE_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}
          else
            docker service update --with-registry-auth \
            --image ${AZURE_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}
          fi
          '''
        // final tests in production
       /* checkout scm
        sh '''
          sleep 60s 
          for i in `seq 1 20`;
          do
            STATUS=$(docker service inspect --format '{{ .UpdateStatus.State }}' ${IMAGE_NAME})
            if [[ "$STATUS" != "updating" ]]; then
              docker run --rm -v ${WORKSPACE}:/go/src/${IMAGE_NAME} -e SERVER=${IMAGE_NAME} golang go test ${IMAGE_NAME} -v --run Integration
              break
            fi
            sleep 10s
          done
          
        '''
       */ 
      }catch(e) {
        sh "docker service update --rollback ${IMAGE_NAME}"
        error "Service update failed in production"
      }finally {
      // sh "docker ps -aq | xargs docker rm || true"
      }
    }
  }