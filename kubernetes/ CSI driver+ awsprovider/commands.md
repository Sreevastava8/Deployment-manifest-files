commnds to install CSI DRIVER with AWS provider using helm after EKS is created

Installation commands we’ll use

helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts

helm repo update

helm install csi-secrets-store \
  secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true \
  --set enableSecretRotation=true



  Then install the AWS provider:

  helm repo add aws-secrets-manager \
  https://aws.github.io/secrets-store-csi-driver-provider-aws

helm repo update

helm install secrets-provider-aws \
  aws-secrets-manager/secrets-store-csi-driver-provider-aws \
  --namespace kube-system


  after installation 

  Verification commands

kubectl get pods -n kube-system

kubectl get pods -n kube-system \
  -l app=secrets-store-csi-driver

  kubectl get pods -n kube-system \
  -l app=secrets-store-csi-driver-provider-aws