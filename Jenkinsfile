properties([
    disableConcurrentBuilds(),
    parameters([
        choice(
            name: 'CLOUD_PROVIDER',
            choices: ['aws', 'gcp'],
            description: 'Choose cloud provider'
        ),
        string(
            name: 'NAMESPACE',
            defaultValue: 'default',
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
  containers:
    - name: tools
      image: google/cloud-sdk:latest
      command: ["sleep"]
      args: ["999999"]
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
            apt-get install -y curl unzip git

            curl -LO https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl
            chmod +x kubectl
            mv kubectl /usr/local/bin/

            kubectl version --client
            '''

            if (params.CLOUD_PROVIDER == "aws") {
                sh '''
                curl https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip
                unzip awscliv2.zip
                ./aws/install || true
                aws --version
                '''
            }

            if (params.CLOUD_PROVIDER == "gcp") {
                sh '''
                gcloud components install gke-gcloud-auth-plugin -q || true
                gcloud version
                '''
            }
        }
    }

    stage('Configure Cluster Access') {
        container('tools') {
            script {

                if (params.CLOUD_PROVIDER == "aws") {
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds'
                    ]]) {

                        sh '''
                        aws eks update-kubeconfig \
                          --region ap-southeast-1 \
                          --name hello-cluster

                        kubectl get nodes
                        '''
                    }
                }

                if (params.CLOUD_PROVIDER == "gcp") {
                    withCredentials([
                        file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')
                    ]) {

                        sh '''
                        export GOOGLE_APPLICATION_CREDENTIALS=$GCP_KEY

                        gcloud auth activate-service-account \
                          --key-file=$GOOGLE_APPLICATION_CREDENTIALS

                        gcloud config set project gke-qa2-36938

                        gcloud container clusters get-credentials \
                          gke-qa2-sg1 \
                          --zone asia-southeast1 \
                          --project gke-qa2-36938 \
                          --internal-ip

                        kubectl get nodes
                        '''
                    }
                }
            }
        }
    }

    stage('Deploy/Delete Application') {
        container('tools') {
            script {

                if (params.ACTION == "deploy") {

                    sh """
                    echo "===== Deploying Application ====="

                    kubectl apply -n ${params.NAMESPACE} -f k8s/deployment.yaml
                    kubectl apply -n ${params.NAMESPACE} -f k8s/service.yaml

                    kubectl scale deployment hello-app \
                      --replicas=1 -n ${params.NAMESPACE}

                    echo "===== Waiting for rollout ====="

                    kubectl rollout status deployment hello-app \
                      -n ${params.NAMESPACE} --timeout=180s

                    kubectl wait --for=condition=available \
                      deployment/hello-app \
                      -n ${params.NAMESPACE} --timeout=180s
                    """

                    // ✅ Only after success
                    scaleDownOtherCloud()
                }

                if (params.ACTION == "delete") {
                    sh """
                    kubectl delete -n ${params.NAMESPACE} -f k8s/deployment.yaml || true
                    kubectl delete -n ${params.NAMESPACE} -f k8s/service.yaml || true
                    """
                }
            }
        }
    }

    stage('Deploy Router (GCP Only)') {
        container('tools') {
            script {

                if (params.ACTION == "deploy" && params.CLOUD_PROVIDER == "gcp") {

                    sh """
                    echo "===== Deploying NGINX Router ====="

                    kubectl apply -n ${params.NAMESPACE} \
                      -f k8s/nginx-router-config.yaml

                    kubectl apply -n ${params.NAMESPACE} \
                      -f k8s/nginx-router-deployment.yaml

                    kubectl rollout status deployment nginx-router \
                      -n ${params.NAMESPACE} --timeout=120s

                    kubectl apply -n ${params.NAMESPACE} \
                      -f k8s/nginx-router-service.yaml

                    kubectl get svc nginx-router-service \
                      -n ${params.NAMESPACE}
                    """
                }

                if (params.ACTION == "delete" && params.CLOUD_PROVIDER == "gcp") {
                    sh """
                    kubectl delete -n ${params.NAMESPACE} \
                      -f k8s/nginx-router-deployment.yaml || true

                    kubectl delete -n ${params.NAMESPACE} \
                      -f k8s/nginx-router-service.yaml || true

                    kubectl delete -n ${params.NAMESPACE} \
                      -f k8s/nginx-router-config.yaml || true
                    """
                }
            }
        }
    }
}
}


// ================= SCALE DOWN LOGIC =================

def scaleDownOtherCloud() {

    // ===== Deploying to AWS → scale down GCP =====
    if (params.CLOUD_PROVIDER == "aws") {

        withCredentials([
            file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')
        ]) {

            sh """
            echo "===== Scaling down GCP AFTER SUCCESS ====="

            export GOOGLE_APPLICATION_CREDENTIALS=$GCP_KEY

            gcloud auth activate-service-account \
              --key-file=$GOOGLE_APPLICATION_CREDENTIALS

            gcloud config set project gke-qa2-36938

            gcloud container clusters get-credentials \
              gke-qa2-sg1 \
              --zone asia-southeast1 \
              --project gke-qa2-36938 \
              --internal-ip

            kubectl scale deployment hello-app \
              --replicas=0 -n ${params.NAMESPACE} || true
            """
        }
    }

    // ===== Deploying to GCP → scale down AWS =====
    if (params.CLOUD_PROVIDER == "gcp") {

        withCredentials([[
            $class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'aws-creds'
        ]]) {

            sh """
            echo "===== Scaling down AWS AFTER SUCCESS ====="

            aws eks update-kubeconfig \
              --region ap-southeast-1 \
              --name hello-cluster

            kubectl get nodes

            kubectl scale deployment hello-app \
              --replicas=0 -n ${params.NAMESPACE} || true
            """
        }
    }
}