properties([
    parameters([

        choice(
            name: 'CLOUD_PROVIDER',
            choices: ['gcp'],
            description: 'Choose cloud provider'
        ),

        string(
            name: 'NAMESPACE',
            defaultValue: 'test-app',
            description: 'Kubernetes namespace'
        ),

        choice(
            name: 'ACTION',
            choices: ['deploy', 'delete'],
            description: 'Choose action'
        )
    ])
])

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

                    gcloud config set project gke-qa2-36938

                    gcloud container clusters get-credentials gke-qa2-sg1 --region asia-southeast1 --project gke-qa2-36938 --internal-ip
                    '''
                }
            }
        }

        stage('Deploy Application') {
            container('tools') {
                def ACTION = params.ACTION

                def commitMsg = currentBuild.changeSets
                    .collect { it.items }
                    .flatten()
                    .collect { it.msg }
                    .join("\n")
                    .trim()
                    .toLowerCase()

                echo "Commit Message: ${commitMsg}"

                // 🔥 Only override for SCM trigger (no user input)
                if (!env.BUILD_USER) {
                    ACTION = commitMsg.contains("delete") ? "delete" : "deploy"
                }

                echo "Final ACTION: ${ACTION}"
                
                if (ACTION == "deploy") {
                    sh """
                    echo "===== Applying ConfigMap ====="
                    
                    kubectl apply -n test-app \
                      -f k8s/configmap-gcp.yaml
                    
                    echo "===== Deploying to GKE ====="

                    kubectl apply -n ${params.NAMESPACE} \
                        -f k8s/deployment.yaml

                    kubectl apply -n ${params.NAMESPACE} \
                        -f k8s/service.yaml

                    echo "===== Pods ====="

                    kubectl get pods -n ${params.NAMESPACE}

                    echo "===== Services ====="

                    kubectl get svc -n ${params.NAMESPACE}
                    """
                }
                if (ACTION == "delete") {

                    sh """
                    echo "===== Deleting from GKE ====="

                    kubectl delete -n ${params.NAMESPACE} \
                        -f k8s/deployment.yaml || true

                    kubectl delete -n ${params.NAMESPACE} \
                        -f k8s/service.yaml || true
                    """
                }
            }
        }
    }
}
