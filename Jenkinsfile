def kubeconfigPath = '/root/.kube/config'
def sonarProjectKey = 'bugbunny_bugbunnyname'
def sonarOrganization = 'bugbunny'
def sonarHostUrl = 'https://sonarcloud.io'
def sonarLogin = '28be4743bcd59168fc9adeb8a386bea7e3d9e9cd'
def snykTokenCredentialId = 'SNYK_TOKEN'
def dockerRegistryUrl = 'https://index.docker.io/v1/'
def dockerRegistryCredentialId = 'DockerLogin'
def ecrRegistryUrl = 'https://315308560490.dkr.ecr.ap-southeast-2.amazonaws.com'
def ecrRegistryCredentialId = 'ecr:ap-southeast-2:0001'
def clusterName = 'TestCube-cluster'
def clusterRegion = 'ap-southeast-2'
def nodegroupName = 'linux-nodes'
def nodegroupType = 't2.micro'
def nodegroupCount = 2
def namespace = 'devsecops'

pipeline {
    agent {
        label 'ServiceNode_1'
    }
    tools { 
        maven 'mavan_3_5_2'  
    }
    stages {
        stage('Compile and Run Sonar Analysis') {
            steps {    
                sh "mvn clean verify sonar:sonar -Dsonar.projectKey=${sonarProjectKey} -Dsonar.organization=${sonarOrganization} -Dsonar.host.url=${sonarHostUrl} -Dsonar.login=${sonarLogin}"
            }
        }

        stage('Run SCA Analysis Using Snyk') {
            steps {        
                withCredentials([string(credentialsId: snykTokenCredentialId, variable: 'SNYK_TOKEN')]) {
                    sh 'mvn snyk:test -fn'
                }
            }
        }   

        stage('Build Docker Image') { 
            steps { 
                withDockerRegistry([credentialsId: dockerRegistryCredentialId, url: dockerRegistryUrl]) {
                    script{
                        app =  docker.build("asg")
                    }
                }
            }
        }

        stage('Push Docker image to ECR') {
            steps {
                script{
                    docker.withRegistry(ecrRegistryUrl, ecrRegistryCredentialId) {
                        app.push("latest")
                    }
                }
            }
        }

        stage('Create Kubernetes cluster') {
            steps {
                sh "eksctl create cluster --name ${clusterName} --version 1.23 --region ${clusterRegion} --nodegroup-name ${nodegroupName} --node-type ${nodegroupType} --nodes ${nodegroupCount}"
                sh "kubectl create namespace ${namespace}"
            }
        }

        stage('Kubernetes Deployment') {
            steps {
                withEnv(["KUBECONFIG=${kubeconfigPath}"]) {
                    sh "kubectl delete all --all -n ${namespace}"
                    sh "kubectl apply -f deployment.yaml --namespace=${namespace}"
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
                    sh "zap.sh -cmd -quickurl http://$(kubectl get services/asgbuggy --namespace=${namespace} -o json| jq -r \".status.loadBalancer.ingress[] | .hostname\") -quickprogress -quickout ${WORKSPACE}/zap_report.html"
                    archiveArtifacts artifacts: 'zap_report.html'
                }
            }
        }

        stage('delete Kubernetes cluster') {
            steps {
                sh "eksctl delete cluster --region=${clusterRegion} --name=${clusterName}"
            }
        } 
    }
post{
	cleanws()
}
}
