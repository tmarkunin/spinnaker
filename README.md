#Steps to install Spinnaker on GKE
#based on https://cloud.google.com/solutions/continuous-delivery-spinnaker-kubernetes-engine

#Enable required APIs for your project

gcloud auth login
gcloud config set project PROJECT_ID

gcloud services enable cloudbuild.googleapis.com
gcloud services enable container.googleapis.com
gcloud services enable sourcerepo.googleapis.com 

#Configure identity and access management

gcloud iam service-accounts create  spinnaker-account --display-name spinnaker-account


export SA_EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:spinnaker-account" \
    --format='value(email)')
export PROJECT=$(gcloud info --format='value(config.project)')


gcloud projects add-iam-policy-binding \
    $PROJECT --role roles/storage.admin --member serviceAccount:$SA_EMAIL

gcloud iam service-accounts keys create spinnaker-sa.json --iam-account $SA_EMAIL




#Create GKE cluster
gcloud config set compute/zone us-central1-f

sudo gcloud container clusters create spinnaker-tutorial \
    --machine-type=n1-standard-2


export SA_JSON=$(cat spinnaker-sa.json)
export PROJECT=$(gcloud info --format='value(config.project)'

sudo gcloud container clusters get-credentials spinnaker-tutorial --zone us-central1-f --project $PROJECT


# Download and install Helm

wget https://storage.googleapis.com/kubernetes-helm/helm-v2.10.0-linux-amd64.tar.gz

tar zxfv helm-v2.10.0-linux-amd64.tar.gz

cp linux-amd64/helm .

kubectl create clusterrolebinding user-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
kubectl create serviceaccount tiller --namespace kube-system
kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

#or
kubectl apply -f helm-rbac.yaml


# We'll need helm connected to our Kubernetes cluster

kubectl create clusterrolebinding --clusterrole=cluster-admin --serviceaccount=default:default spinnaker-admin

helm init --service-account=tiller

helm update


#Then we'll want to install prometheus in order to collect metrics for our Canary efforts. GKE default storage class #is 'standard'
helm install --name prometheus --set server.persistentVolume.storageClass=standard stable/prometheus

#Connect to Prometheus
#Get the Prometheus server URL by running these commands in the same shell:
#  export POD_NAME=$(kubectl get pods --namespace default -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace default port-forward $POD_NAME 9090

  helm status prometheus

  #Create the file (spinnaker-config.yaml) describing the configuration for how Spinnaker should be installed

)


