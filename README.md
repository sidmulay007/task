pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentails"(enter dockerhub id)"
        GKE_DEPLOYMENT = "(enter deployment)"
        GKE_CLUSTER_NAME = "(enter gke name)"
        GKE_NAMESPACE = "(enter namespace)"
        EMAIL_RECIPIENT = "(enter your email)"
    }
    stages {
        stage('Git Checkout') {
            steps {
                git credentialID: (mention ID), url: (menton URL of gitrepo)
                echo 'git checkout doen'
            }
        }
        stage('Build') {
            steps {
                script {
                    withMaven(maven: MAVEN_HOME, mavenLocalRepo: '(mention repo eg. .repo)') {
                        sh 'mvn clean package'
                   }
                }
            }
        }
        stage('SonarQube Scan') {
        environment {
        SONARQUBE_SCANNER_HOME = tool 'SonarQubeScanner'
            steps {
                script {
                    withSonarQubeEnv('YourSonarQubeServer', credentialID: 'sonar-token') {
                        sh "${SONARQUBE_SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectkey=(mention project key)"
                    }
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
                }
        }
    post {
        success {
            echo 'Quality Gate passed!'
        }
        failure {
            echo 'Quality Gate failed.'
        }
    }
        stage('Build Docker Image') {
            steps {
                    sh 'sudo docker build -t dockerhubusername/dockerhubreponame:$BUILD_NUMBER .'
            }
        }
         stage('Login to Dockerhub') {
            steps {
                    sh 'echo $DOCKERHUB_CREDENTIAL_PWD|sudo docker login -u $DOCKERHUB_CREDENTIALS_USR'
            }
        }
         stage('Pushing image to dockerhub') {
            steps {
                    sh 'sudo docker push (dockerhubusername)/(dockerhubreponame):$BUILD_NUMBER .'
            }
        }
         post{
            always{
                    sh 'docker logout'
                  }
        }
        stage('Deploy to GKE') {
            steps {
                    withCredentials([gkeAccount(credentialsId: '(mention gke credentials id)', variable: 'GKE_CREDS')]) {
                        sh 'gcloud auth activate-service-account --key-file=$GKE_CREDS'
                        sh 'gcloud container clusters get-credentials $GKE_CLUSTER_NAME --region=your-gke-region'
                    }
                    sh 'kubectl set image deployment/$GKE_DEPLOYMENT your-container=$DOCKER_REGISTRY/your-image:latest -n $GKE_NAMESPACE'
                }
            }
        }

        stage('Autoscaling') {
            steps {
                script {
                    //(adjust the command based on your metrics)
                    def currentCPU = sh(script: 'kubectl top pods --namespace=$GKE_NAMESPACE | grep your-pod-name | awk \'{print $2}\'', returnStdout: true).trim().toInteger()

                    if (currentCPU >= 70) {
                        // Scale up if CPU utilization is >= 70%
                        sh 'kubectl scale deployment $GKE_DEPLOYMENT --replicas=3 -n $GKE_NAMESPACE'
                    } else if (currentCPU < 15) {
                        // Scale down if CPU utilization is < 15%
                        sh 'kubectl scale deployment $GKE_DEPLOYMENT --replicas=1 -n $GKE_NAMESPACE'
                    }
                }
            }
        }
    }

    post {
        success {
            // Notification on success
            emailext attachLog: true, body: "Deployment successful", subject: "Jenkins Pipeline - Deployment Success", to: "$EMAIL_RECIPIENT"
            slackSend(color: 'good', message: 'Deployment successful')
        }

        failure {
            // Notification on failure
            emailext attachLog: true, body: "Deployment failed. Check Jenkins logs for details.", subject: "Jenkins Pipeline - Deployment Failure", to: "$EMAIL_RECIPIENT"
            slackSend(color: 'danger', message: 'Deployment failed. Check Jenkins logs for details.')
        }
    }
}