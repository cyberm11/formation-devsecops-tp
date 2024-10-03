@Library('slack') _

/////// ******************************* Code for fectching Failed Stage Name ******************************* ///////
import io.jenkins.blueocean.rest.impl.pipeline.PipelineNodeGraphVisitor
import io.jenkins.blueocean.rest.impl.pipeline.FlowNodeWrapper
import org.jenkinsci.plugins.workflow.support.steps.build.RunWrapper
import org.jenkinsci.plugins.workflow.actions.ErrorAction

// Get information about all stages, including the failure cases
// Returns a list of maps: [[id, failedStageName, result, errors]]
@NonCPS
List<Map> getStageResults(RunWrapper build) {
  // Get all pipeline nodes that represent stages
  def visitor = new PipelineNodeGraphVisitor(build.rawBuild)
  def stages = visitor.pipelineNodes.findAll { it.type == FlowNodeWrapper.NodeType.STAGE }

  return stages.collect { stage ->
        // Get all the errors from the stage
        def errorActions = stage.getPipelineActions(ErrorAction)
        def errors = errorActions?.collect { it.error }.unique()

        return [
            id: stage.id,
            failedStageName: stage.displayName,
            result: "${stage.status.result}",
            errors: errors
        ]
  }
}
// Get information of all failed stages not ok
@NonCPS
List<Map> getFailedStages(RunWrapper build) {
  return getStageResults(build).findAll { it.result == 'FAILURE' }
}


pipeline {
  agent any
  
  stages {
    //--------------------------
      stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar' //so that they can be downloaded later
            }
        } 
        //--------------------------
    stage('UNIT test & jacoco ') {
      steps {
        sh "mvn test"
      }
 
  post {

        always {

          junit 'target/surefire-reports/*.xml'

          jacoco execPattern: 'target/jacoco.exec'

        }

      }
    }
    //--------------------------

    stage('SonarQube - SAST') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          sh "sonar-scanner \
  -Dsonar.projectKey=maven-jenkins-pipeline2 \
  -Dsonar.host.url=http:192.168.56.101:9999 \
  -Dsonar.login=0dd0f00e0f52c397d2ed0174b77c4bdd23c73efb"
        }
      }
    }

    //--------------------------
    stage('Docker Build and Push') {
      steps {
        withCredentials([string(credentialsId: 'svc-all', variable: 'DOCKER_HUB_PASSWORD')]) {
          sh 'sudo docker login -u cyberm11  -p $DOCKER_HUB_PASSWORD'
          sh 'printenv'
          sh 'sudo docker build -t cyberm11/devops-app:""$GIT_COMMIT"" .'
          sh 'sudo docker push cyberm11/devops-app:""$GIT_COMMIT""'
        }
 
      }
    }
	  //-----------------------------------

	  //     	 stage('Vulnerability Scan - Docker Trivy') {
    //    steps {
	  //       withCredentials([string(credentialsId: 'svc-all', variable: 'TOKEN')]) {
		// 	 catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
    //              sh "sed -i 's#token_github#${TOKEN}#g' trivy-image-scan.sh"
    //              sh "sudo bash trivy-image-scan.sh"
	  //      }
		// }
    //    }
    //  }
	  
//-----------------------
	  	  

    stage('Vulnerability Scan - Kubernetes') {
      steps {
        parallel(
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          "OPA Scan": {
            sh 'sudo docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
          },
          "Kubesec Scan": {
            sh "sudo bash kubesec-scan.sh"
          },
          "Trivy Scan": {
            sh "sudo bash trivy-k8s-scan.sh"
          }
            }

        )
      
    }
}
    //----------------------------

        stage('Deployment Kubernetes  ') {
      steps {
        withKubeConfig([credentialsId: 'svc-all']) {
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
              sh "sed -i 's#replace#cyberm11/devops-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
              sh 'kubectl apply -f k8s_deployment_service.yaml'
            }
        }
      }
    }

//---------------------------------
}

post {
        success {
      script {
        sendNotification('SUCCESS')
      }
        }
        failure {
      script {
        sendNotification('FAILURE')
      }
        }
        unstable {
      script {
        sendNotification('UNSTABLE')
      }
        }
    }
}