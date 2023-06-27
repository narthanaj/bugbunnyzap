def kubeconfigPath = '/root/.kube/config'

pipeline {
      agent {
            label 'ServiceNode_1'
            }
  tools { 
        maven 'mavan_3_5_2'  
    }
   stages{
    stage('Compile and Run Sonar Analysis') {
            steps {	
		sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=bugbunny_bugbunnyname -Dsonar.organization=bugbunny -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=28be4743bcd59168fc9adeb8a386bea7e3d9e9cd'
			}
    }

	stage('Run SCA Analysis Using Snyk') {
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

	stage('Create Kubernetes cluster') {
	    steps {
	        sh('eksctl create cluster --name TestCube-cluster --version 1.23 --region ap-southeast-2 --nodegroup-name linux-nodes --node-type t2.micro --nodes 2')
	    }
	}

	stage('Kubernetes Deployment') {
	    steps {
	        withEnv(["KUBECONFIG=${kubeconfigPath}"]) {
	            sh('kubectl delete all --all -n devsecops')
	            sh('kubectl apply -f deployment.yaml --namespace=devsecops')
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
	        withEnv(["KUBECONFIG=${kubeconfigPath}"]) {
	            sh('zap.sh -cmd -quickurl http://$(kubectl get services/asgbuggy --namespace=devsecops -o json| jq -r ".status.loadBalancer.ingress[] | .hostname") -quickprogress -quickout ${WORKSPACE}/zap_report.html')
	            archiveArtifacts artifacts: 'zap_report.html'
	        }
	    }
	}


	stage('delete Kubernetes cluster') {
	    steps {
	        sh('eksctl delete cluster --region=ap-southeast-2 --name=TestCube-cluster')
	    }
	} 
  }
}
