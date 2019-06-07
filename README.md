#Steps to install Spinnaker on GKE
#based on https://cloud.google.com/solutions/continuous-delivery-spinnaker-kubernetes-engine

#Enable required APIs for your project
gcloud auth login
gcloud auth application-default login
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

export SA_JSON=$(cat spinnaker-sa.json)


#Create GKE cluster
gcloud config set compute/zone us-central1-f

sudo gcloud container clusters create spinnaker-tutorial \
    --machine-type=n1-standard-2


sudo gcloud container clusters get-credentials spinnaker-tutorial --zone us-central1-f --project $PROJECT


# Download and install Helm

wget https://storage.googleapis.com/kubernetes-helm/helm-v2.14.0-linux-amd64.tar.gz

tar zxfv helm-v2.14.0-linux-amd64.tar.gz

cp linux-amd64/helm .

chmod +x helm

#update tiller version if required --  sudo ~/helm init --upgrade


sudo kubectl create clusterrolebinding user-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
sudo kubectl create serviceaccount tiller --namespace kube-system
sudo kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

#or
kubectl apply -f helm-rbac.yaml


# We'll need helm connected to our Kubernetes cluster

sudo kubectl create clusterrolebinding --clusterrole=cluster-admin --serviceaccount=default:default spinnaker-admin

sudo ./helm init --service-account=tiller

sudo ./helm update


#Then we'll want to install prometheus in order to collect metrics for our Canary efforts. GKE default storage class #is 'standard'
sudo ~/helm install --name prometheus --set server.persistentVolume.storageClass=standard stable/prometheus

#Connect to Prometheus
#Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace default -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace default port-forward $POD_NAME 9090

  helm status prometheus

  #We also need to create a secret for Spinnaker from our Kubeconfig file in order for Spinnaker to be able to talk to our cluster (for deploying applications)
   sudo kubectl create secret generic kubeconfig --from-file=/home/vagrant/.kube/config -n spinnaker


  #create gcp static ip for Ingress service. We will <static-ip>.nip.io - Doesn't work

  gcloud compute addresses create spinnaker-static-ip --global

  export DOMAIN=<static-ip>


  #Create the file (spinnaker-config.yaml) describing the configuration for how Spinnaker should be installed

 git clone https://github.com/tmarkunin/spinnaker.git


#Finally, we can launch Helm to install spinnaker
  sudo ~/helm install -f spinnaker/spinnaker-config.yaml --name cd --timeout 1200 --namespace spinnaker stable/spinnaker

  #configure spinnaker
  

 #open spinnaker ports
 gcloud compute firewall-rules create spinnaker-ui --allow tcp:9000 --source-ranges=0.0.0.0/0
 

 sudo kubectl exec --namespace spinnaker -it cd-spinnaker-halyard-0 bash

 #add spinnaker ingress

 sudo kubectl apply -f spinnaker/spinnaker-ingress.yaml



  #delete spinnaker 
  sudo ~/helm del --purge cd





