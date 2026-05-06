podTemplate(
    yaml: '''
apiVersion: v1
kind: Pod
spec:

  tolerations:
    - key: "role"
      operator: "Exists"
      effect: "NoSchedule"
    - key: "CriticalAddonsOnly"
      operator: "Exists"

  containers:
    - name: tools
      image: google/cloud-sdk:latest
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

        stage('Setup Tools') {
            container('tools') {
                sh '''
                # Install kubectl (if not already present)
                apt-get update
                apt-get install -y curl

                curl -LO "https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl"
                chmod +x kubectl
                mv kubectl /usr/local/bin/

                # Ensure GKE auth plugin
                gcloud components install gke-gcloud-auth-plugin -q || true

                kubectl version --client
                gcloud version
                '''
            }
        }

        stage('Authenticate + Configure GKE') {
            container('tools') {
                withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {
                    sh '''
                    export GOOGLE_APPLICATION_CREDENTIALS=$GCP_KEY

                    gcloud auth activate-service-account \
                      --key-file=$GOOGLE_APPLICATION_CREDENTIALS

                    gcloud config set projectgke-qa2-36938

                    gcloud container clusters get-credentials \
                      gke-qa2-sg1 \
                      --zone asia-southeast1 \
                      --project gke-qa2-36938

                    kubectl get nodes
                    '''
                }
            }
        }

        stage('Deploy Application') {
            container('tools') {
                sh '''
                kubectl apply -n test-app -f k8s/deployment.yaml
                kubectl apply -n test-app -f k8s/service.yaml

                kubectl get pods -n test-app
                kubectl get svc -n test-app
                '''
            }
        }
    }
}