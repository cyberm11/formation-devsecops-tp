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

    stage('SonarQube_scanne') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          withCredentials([string(credentialsId: 'svc-all', variable: 'TOKEN')]){
          sh "mvn sonar:sonar \
  -Dsonar.projectKey=maven-jenkins-pipeline2 \
  -Dsonar.host.url=http://192.168.56.101:9999 \
  -Dsonar.login=${TOKEN}"
        }
      }}
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

	      	 stage('Vulnerability Scan - Docker Trivy') {
       steps {
	        withCredentials([string(credentialsId: 'svc-all', variable: 'TOKEN')]) {
			 catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                 sh "sed -i 's#token_github#${TOKEN}#g' trivy-image-scan.sh"
                 sh "sudo bash trivy-image-scan.sh"
	       }
		}
       }
     }
	  
//-----------------------
	  	  

    stage('Vulnerability Scan - Kubernetes') {
      steps {
        parallel(
          "OPA Scan": {
            sh 'sudo docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
          },
          "Kubesec Scan": {
            sh "sudo bash kubesec-scan.sh"
          },
          "Trivy Scan": {
            sh "sudo bash trivy-k8s-scan.sh"
          }

        )
      
    }
}
    //----------------------------

        stage('Deployment Kubernetes  ') {
      steps {
        withKubeConfig([credentialsId: 'svc-all']) {
              sh "sed -i 's#replace#cyberm11/devops-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
              sh 'kubectl apply -f k8s_deployment_service.yaml'
        }
      }
    }

//---------------------------------
}
}