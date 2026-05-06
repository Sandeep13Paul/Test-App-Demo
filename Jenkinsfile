podTemplate(
    yaml: '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: tools
      image: ubuntu:22.04
      command:
        - sleep
      args:
        - "999999"
      tty: true
'''
) {

    node(POD_LABEL) {

        stage('Checkout') {
            checkout scm
        }

        stage('Install Tools') {
            container('tools') {
                sh '''
                apt-get update

                apt-get install -y \
                  curl \
                  unzip \
                  git \
                  apt-transport-https \
                  ca-certificates \
                  gnupg

                # Install kubectl
                curl -LO "https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl"
                chmod +x kubectl
                mv kubectl /usr/local/bin/

                # Install gcloud SDK
                curl https://sdk.cloud.google.com | bash

                export PATH=$PATH:/root/google-cloud-sdk/bin

                gcloud version
                kubectl version --client
                '''
            }
        }

        stage('Authenticate GCP') {
            container('tools') {
                withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {
                    sh '''
                    export GOOGLE_APPLICATION_CREDENTIALS=$GCP_KEY

                    /root/google-cloud-sdk/bin/gcloud auth activate-service-account \
                      --key-file=$GOOGLE_APPLICATION_CREDENTIALS

                    /root/google-cloud-sdk/bin/gcloud config set project YOUR_PROJECT_ID
                    '''
                }
            }
        }

        stage('Configure GKE Access') {
            container('tools') {
                withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {
                    sh '''
                    export GOOGLE_APPLICATION_CREDENTIALS=$GCP_KEY

                    /root/google-cloud-sdk/bin/gcloud container clusters get-credentials \
                      YOUR_CLUSTER_NAME \
                      --zone YOUR_ZONE \
                      --project YOUR_PROJECT_ID

                    kubectl get nodes
                    '''
                }
            }
        }

        stage('Deploy Application') {
            container('tools') {
                sh '''
                kubectl apply -n default -f k8s/deployment.yaml
                kubectl apply -n default -f k8s/service.yaml

                kubectl get pods -n default
                kubectl get svc -n default
                '''
            }
        }
    }
}