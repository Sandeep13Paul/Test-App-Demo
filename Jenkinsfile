properties([
    disableConcurrentBuilds(),
    parameters([
        choice(name: 'CLOUD_PROVIDER', choices: ['aws', 'gcp'], description: 'Choose cloud provider'),
        string(name: 'NAMESPACE', defaultValue: 'default', description: 'Kubernetes namespace'),
        choice(name: 'ACTION', choices: ['deploy', 'delete'], description: 'Choose action')
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


            sh '''
                curl https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip
                unzip awscliv2.zip
                ./aws/install || true
            '''



            sh '''
                gcloud components install gke-gcloud-auth-plugin -q || true
            '''
        }
    }

    stage('Configure Cluster Access') {
        container('tools') {
            script {

                if (params.CLOUD_PROVIDER == "aws") {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                        sh '''
                        aws eks update-kubeconfig --region ap-southeast-1 --name hello-cluster
                        kubectl get nodes
                        '''
                    }
                }

                if (params.CLOUD_PROVIDER == "gcp") {
                    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {
                        sh '''
                        export GOOGLE_APPLICATION_CREDENTIALS=$GCP_KEY
                        gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                        gcloud config set project gke-qa2-36938
                        gcloud container clusters get-credentials gke-qa2-sg1 \
                          --zone asia-southeast1 --project gke-qa2-36938 --internal-ip
                        kubectl get nodes
                        '''
                    }
                }
            }
        }
    }

    stage('Deploy Application') {
        container('tools') {
            script {

                // ================= AWS =================
                if (params.CLOUD_PROVIDER == "aws" && params.ACTION == "deploy") {

                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {

                        sh '''
                        aws eks update-kubeconfig --region ap-southeast-1 --name hello-cluster
                        '''

                        sh """
                        kubectl apply -n ${params.NAMESPACE} -f k8s/deployment.yaml
                        kubectl apply -n ${params.NAMESPACE} -f k8s/service.yaml

                        kubectl rollout status deployment hello-app -n ${params.NAMESPACE} --timeout=180s
                        kubectl wait --for=condition=available deployment/hello-app -n ${params.NAMESPACE} --timeout=180s
                        """

                        scaleDownGCP()
                    }
                }

                // ================= GCP =================
                if (params.CLOUD_PROVIDER == "gcp" && params.ACTION == "deploy") {

                    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {

                        sh '''
                        export GOOGLE_APPLICATION_CREDENTIALS=$GCP_KEY
                        gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                        gcloud config set project gke-qa2-36938
                        gcloud container clusters get-credentials gke-qa2-sg1 \
                          --zone asia-southeast1 --project gke-qa2-36938 --internal-ip
                        '''

                        sh """
                        kubectl apply -n ${params.NAMESPACE} -f k8s/deployment.yaml
                        kubectl apply -n ${params.NAMESPACE} -f k8s/service.yaml

                        kubectl rollout status deployment hello-app -n ${params.NAMESPACE} --timeout=180s
                        kubectl wait --for=condition=available deployment/hello-app -n ${params.NAMESPACE} --timeout=180s
                        """

                        scaleDownAWS()
                    }
                }

                // ===== AWS DELETE =====
                if (params.CLOUD_PROVIDER == "aws") {

                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds'
                    ]]) {

                        sh """
                        echo "===== Deleting from AWS ====="

                        aws eks update-kubeconfig \
                        --region ap-southeast-1 \
                        --name hello-cluster

                        kubectl delete -n ${params.NAMESPACE} \
                        -f k8s/deployment.yaml || true

                        kubectl delete -n ${params.NAMESPACE} \
                        -f k8s/service.yaml || true
                        """
                    }
                }

                // ===== GCP DELETE =====
                if (params.CLOUD_PROVIDER == "gcp") {

                    withCredentials([
                        file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')
                    ]) {

                        sh """
                        echo "===== Deleting from GCP ====="

                        export GOOGLE_APPLICATION_CREDENTIALS=\$GCP_KEY

                        gcloud auth activate-service-account \
                        --key-file=\$GOOGLE_APPLICATION_CREDENTIALS

                        gcloud config set project gke-qa2-36938

                        gcloud container clusters get-credentials gke-qa2-sg1 \
                        --zone asia-southeast1 \
                        --project gke-qa2-36938 \
                        --internal-ip

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

    stage('Deploy Router (GCP Only)') {
        container('tools') {
            script {
                if (params.CLOUD_PROVIDER == "gcp" && params.ACTION == "deploy") {

                    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {

                        sh '''
                        export GOOGLE_APPLICATION_CREDENTIALS=$GCP_KEY
                        gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                        gcloud config set project gke-qa2-36938
                        gcloud container clusters get-credentials gke-qa2-sg1 \
                          --zone asia-southeast1 --project gke-qa2-36938 --internal-ip
                        '''

                        sh """
                        kubectl apply -n ${params.NAMESPACE} -f k8s/nginx-router-config.yaml
                        kubectl apply -n ${params.NAMESPACE} -f k8s/nginx-router-deployment.yaml
                        kubectl rollout status deployment nginx-router -n ${params.NAMESPACE} --timeout=120s
                        kubectl apply -n ${params.NAMESPACE} -f k8s/nginx-router-service.yaml
                        kubectl get svc nginx-router-service -n ${params.NAMESPACE}
                        """
                    }
                }
            }
        }
    }
}
}

// ================= FUNCTIONS =================

def scaleDownAWS() {
    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
        sh """
        echo "Scaling down AWS AFTER success"
        aws eks update-kubeconfig --region ap-southeast-1 --name hello-cluster
        kubectl scale deployment hello-app --replicas=0 -n ${params.NAMESPACE} || true
        """
    }
}

def scaleDownGCP() {
    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {
        sh """
        echo "Scaling down GCP AFTER success"

        export GOOGLE_APPLICATION_CREDENTIALS=\$GCP_KEY

        gcloud auth activate-service-account \
          --key-file=\$GOOGLE_APPLICATION_CREDENTIALS

        gcloud config set project gke-qa2-36938

        gcloud container clusters get-credentials gke-qa2-sg1 \
          --zone asia-southeast1 \
          --project gke-qa2-36938 \
          --internal-ip

        kubectl scale deployment hello-app \
          --replicas=0 \
          -n ${params.NAMESPACE} || true
        """
    }
}