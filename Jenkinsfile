#!groovy
  
def project = 'praveenr'
def appName = 'hello-world-bluegreen'
def feSvcName = "${appName}"
def namespace = 'default'
def imageTag = "quay.io/${project}/${appName}:v${env.BUILD_NUMBER}"
def prevImageTag = ''
def prevBuildNum = ''
def firstDeploy = falsbluegreen

node {
  // Check if there's a previous deployment, if so, get the image version so we can rollback if needed
  try {
    prevImageTag = sh(
      script: "kubectl get deployment hello-world-blue -n ${namespace} -o jsonpath='{.spec.template.spec.containers[0].image}'",
      returnStdout: true
    ).trim()
    echo "Previous Image: ${prevImageTag}"
    prevBuildNum = prevImageTag.split(':')[1]
    echo "Previous Build Version: ${prevBuildNum}"
  } catch (err) {
    echo "No Previous Deployment"
    firstDeploy = true
  }

  checkout scm
  sh("printenv")
	
  stage 'Login to Quay.io'
  sh("docker login -u=\"${env.quay_username}\" -p=\"${env.quay_password}\" quay.io")
	
  stage 'Build image'
  sh("docker build -t ${imageTag} .")

  stage 'Push image to Quay.io registry'
  sh("docker push ${imageTag}")

  // If this is the first deployment
  if (firstDeploy) {
    stage 'First Deployment'
    // Update images in manifests with current build
    sh("sed -i.bak 's#quay.io/${project}/${appName}:.*\$#${imageTag}#' ./k8s/blue/*.yaml")
    sh("sed -i.bak 's#quay.io/${project}/${appName}:.*\$#${imageTag}#' ./k8s/green/*.yaml")
    sh("kubectl --namespace=${namespace} apply -f k8s/services/")
    sh("kubectl --namespace=${namespace} apply -f k8s/blue/")
    sh("kubectl --namespace=${namespace} label deployment hello-world-blue --overwrite version=v${BUILD_NUMBER}")
    sh("kubectl --namespace=${namespace} label pod  -l env=blue --all --overwrite version=v${BUILD_NUMBER}")

    sh("kubectl --namespace=${namespace} apply -f k8s/green/")
    sh("kubectl --namespace=${namespace} label deployment hello-world-green --overwrite version=v${BUILD_NUMBER}")
    sh("kubectl --namespace=${namespace} label pod  -l env=green --all --overwrite version=v${BUILD_NUMBER}")
    currentBuild.result = 'SUCCESS'   
    return
  } else {
    // Roll out to blue environment
    stage "Deploy Blue"
    
    // Change deployed image in blue to the one we just built
    sh("kubectl --namespace=${namespace} set image deployment/hello-world-blue hello-world=${imageTag}")
    
    // Apply version label to deployment
    sh("kubectl --namespace=${namespace} label deployment hello-world-blue --overwrite version=v${BUILD_NUMBER}")
    sh("kubectl --namespace=${namespace} label pod  -l env=blue --all --overwrite version=v${BUILD_NUMBER}")
  }
  stage 'Verify Blue'
  def didTimeout = false
  def userInput = true
  try {
    timeout(time:1, unit:'DAYS') {
      userInput = input(id: 'promoteToProd', message: 'Approve rollout to production?')
      echo "userInput: [${userInput}]" 
    }
  } catch(err) { // timeout reached or input false
    stage 'Rolling Back Blue'
    echo "Rollout Aborted"
    echo "userInput: [${userInput}]"
    currentBuild.result = 'FAILURE'

    // If there was a previous deployment, roll it back
    if (prevImageTag != '') {
      echo "Rolling back to: ${prevImageTag}"

      // Change deployed image in blue to the previous image
    	sh("kubectl --namespace=${namespace} set image deployment/hello-world-blue hello-world=${prevImageTag}")	
    	sh("kubectl --namespace=${namespace} label deployment hello-world-blue --overwrite version=${prevBuildNum}")
      sh("kubectl --namespace=${namespace} label pod  -l env=blue --all --overwrite version=${prevBuildNum}")

    }
    error('Aborted')
  }

  if (!firstDeploy) {
  stage 'Rollout to Production' 
    // Roll out to production environment
    // Change deployed image in blue to the one we just built
    //sh("sed -i.bak 's#quay.io/${project}/${appName}:.*\$#${imageTag}#' ./k8s/green/*.yaml")
    //sh("kubectl --namespace=${namespace} apply -f k8s/green/")
    sh("kubectl --namespace=${namespace} set image deployment/hello-world-green hello-world=${imageTag}")
    sh("kubectl --namespace=${namespace} label deployment hello-world-green --overwrite version=v${BUILD_NUMBER}")
    sh("kubectl --namespace=${namespace} label pod  -l env=green --all --overwrite version=v${BUILD_NUMBER}")
    currentBuild.result = 'SUCCESS'
  }
}
