pipeline {
      agent {
            label 'ServiceNode_1'
            }
  tools { 
        maven 'mavan_3_5_2'  
    }
   stages{
    stage('CompileandRunSonarAnalysis') {
            steps {	
		sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=bugbunny_bugbunnyname -Dsonar.organization=bugbunny -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=28be4743bcd59168fc9adeb8a386bea7e3d9e9cd'
			}
    }

	stage('RunSCAAnalysisUsingSnyk') {
            steps {		
				withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
					sh 'mvn snyk:test -fn'
				}
			}
    }	

	stage('Build Docker Image') { 
            steps { 
               withDockerRegistry([credentialsId: "DockerLogin", url: "https://index.docker.io/v1/"]) {
                 script{
                 app =  docker.build("asg")
                 }
               }
            }
    }

	stage('Push Docker image to ECR') {
            steps {
                script{
                    docker.withRegistry('https://315308560490.dkr.ecr.ap-southeast-2.amazonaws.com', 'ecr:ap-southeast-2:0001') {
                    app.push("latest")
                    }
                }
            }
    	}
	   
	stage('Kubernetes Deployment of ASG Bugg Web Application') {
	   steps {
	      withKubeConfig([credentialsId: 'kubelogin']) {
		  sh('kubectl delete all --all -n devsecops')
		  sh ('kubectl apply -f deployment.yaml --namespace=devsecops')
		}
	      }
   	}
	   
	stage ('wait_for_testing'){
	   steps {
		   sh 'pwd; sleep 180; echo "Application Has been deployed on K8S"'
	   	}
	   }
	   
	stage('RunDASTUsingZAP') {
          steps {
		    withKubeConfig([credentialsId: 'kubelogin']) {
				sh('zap.sh -cmd -quickurl http://$(kubectl get services/asgbuggy --namespace=devsecops -o json| jq -r ".status.loadBalancer.ingress[] | .hostname") -quickprogress -quickout ${WORKSPACE}/zap_report.html')
				archiveArtifacts artifacts: 'zap_report.html'
		    }
	     }
       } 
  }
}
