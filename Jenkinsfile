pipeline {
  agent any
  tools { 
        maven 'Maven_3_8_4'  
    }
   stages{
    stage('CompileandRunSonarAnalysis') {
            steps {	
		sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=ramgdevops8_devsecops -Dsonar.organization=ramgdevops8 -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=b065ad4ebc7fa86b3d69407eb5bc85289fb8b889'
			}
    }

stage('RunSCAAnalysisUsingSnyk') {
    steps {
        withCredentials([string(credentialsId: 'SNYK_token', variable: 'SNYK_token')]) {
            sh '''
            mvn snyk:test --json > snyk-report.json || true
            '''
            archiveArtifacts 'snyk-report.json'
        }
    }
}

	stage('Build') { 
            steps { 
               withDockerRegistry([credentialsId: "dockerlogin", url: ""]) {
                 script{
                 app =  docker.build("ram_devsecops")
                 }
               }
            }
    }

	stage('Push') {
            steps {
                script{
                    docker.withRegistry('https://261399255188.dkr.ecr.ap-south-1.amazonaws.com/ram_devsecops', 'ecr:ap-south-1:aws-credentials') {
                    app.push("latest")
                    }
                }
            }
    	}
	   
	stage('Kubernetes Deployment of  Bugg Web Application') {
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
            sh '''
            HOST=$(kubectl get svc rambuggy -n devsecops -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

            echo "Scanning: http://$HOST"
            # Remove old reports if they exist
             rm -f "$WORKSPACE/zap_report.html"
             rm -f "$WORKSPACE/zap.yaml"
			 
            docker run --rm \
              --user $(id -u):$(id -g) \
              -v "$WORKSPACE:/zap/wrk" \
              ghcr.io/zaproxy/zaproxy:stable \
              zap-baseline.py \
              -t "http://$HOST" \
              -r zap_report.html \
			  -I || true

			echo "Checking report..."
            ls -lh "$WORKSPACE"
            '''
        }

        archiveArtifacts artifacts: '**/zap_report.html', fingerprint: true, allowEmptyArchive: true
              }
         } 
  }
}
