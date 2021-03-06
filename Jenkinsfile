IMAGETAG = ''
pipeline{
    agent any
    stages{
        stage('Lint HTML, Docker'){
            steps{
              sh  'tidy -q -e public/*.html'
              sh  'hadolint Dockerfile'
            }            
        }

        stage('Install Dependencies'){
             steps{
                 sh 'npm i'
             }   
        }

        stage('Test React App'){
             steps{
                 sh 'npm run test-ci'
             }   
        }

        stage('Build React App'){
             steps{
                 sh 'npm run build'
             }   
        }

        stage('Build & Push Docker'){       
            steps{ 
                withCredentials([usernamePassword(credentialsId: 'docker', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    script{
                        IMAGETAG = sh (script: 'git log -1 --pretty=%H',returnStdout: true).trim() 
                    }
                    echo "Git commit id: ${IMAGETAG}"
                    echo "p ${PASSWORD}, u: ${USERNAME}"
                    sh "docker build -t ujjwaldocker/hello-react:${IMAGETAG} ."

                    sh "docker login -u ${USERNAME} -p ${PASSWORD} && docker push docker.io/ujjwaldocker/hello-react:${IMAGETAG}"
                }   
            }
        }

        stage('Create K8s cluster') {
            environment{
                KOPS_GREEN_CLUSTER_NAME = "greencluster.k8s.local"
                DEPLOYMENT_NAME = "react-app-kubernetes"
                KOPS_STATE_STORE = "s3://greencluster-for-reactapp-state-store"
            }
            steps {
                withAWS(credentials:'jenkins-aws'){    
                    sh 'kops version'
                    sh 'kops update cluster --name ${KOPS_GREEN_CLUSTER_NAME} --yes'
                }
            }
        }
        
        stage('Validating K8s cluster') {
            environment{
                KOPS_GREEN_CLUSTER_NAME = "greencluster.k8s.local"
                DEPLOYMENT_NAME = "react-app-kubernetes"
                KOPS_STATE_STORE = "s3://greencluster-for-reactapp-state-store"
            }
            options {
                timeout(time: 300, unit: 'MINUTES') 
            }            
            steps {
                withAWS(credentials:'jenkins-aws'){
                    sh "kops validate cluster"
                }
            }
        }

        stage('Deploy on AWS & Expose service'){
            environment{
                KOPS_GREEN_CLUSTER_NAME = "greencluster.k8s.local"
                DEPLOYMENT_NAME = "react-app-kubernetes"
                KOPS_STATE_STORE = "s3://greencluster-for-reactapp-state-store"
                IMAGETAG = sh (script: 'git log -1 --pretty=%H',returnStdout: true).trim() 
            }
            steps{
               withAWS(credentials:'jenkins-aws'){ 
                    sh "echo ${IMAGETAG}"
                    sh "kubectl create deployment ${DEPLOYMENT_NAME} --image=docker.io/ujjwaldocker/hello-react:${IMAGETAG}"
                    sh 'kubectl expose deployment/${DEPLOYMENT_NAME} --type="NodePort" --port 80 --type=LoadBalancer'
                    sh 'export NODE_PORT=$(kubectl get services/${DEPLOYMENT_NAME} -o go-template="{{(index .spec.ports 0).nodePort}}")'
                    sh 'echo NODE_PORT=${NODE_PORT}'
               }
            }            
        }


        stage('Update Blue with New Release'){
            environment{
                KOPS_GREEN_CLUSTER_NAME = "greencluster.k8s.local"
                DEPLOYMENT_NAME = "react-app-kubernetes"
                KOPS_STATE_STORE = "s3://greencluster-for-reactapp-state-store"
                IMAGETAG = sh (script: 'git log -1 --pretty=%H',returnStdout: true).trim() 
            }
            steps{
               input message: 'Finished verification? (Click "Proceed" to continue)'
               sh 'kops version'
               sh 'kubectl config use-context bluecluster.k8s.local'
               sh 'kubectl set image deployments/${DEPLOYMENT_NAME} ${DEPLOYMENT_NAME}=docker.io/ujjwaldocker/hello-react:${IMAGETAG}'
               sh 'kubectl rollout status deployments/kubernetes-bootcamp'
               echo "Deleting Green Environment..."
               sh 'kubectl config use-context ${KOPS_GREEN_CLUSTER_NAME}'
               sh 'kops delete cluster --name ${KOPS_GREEN_CLUSTER_NAME} --yes'
            }   
        }
    }
}