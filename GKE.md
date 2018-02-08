# GKE Jenkins

* [code](https://cloud.google.com/solutions/jenkins-on-container-engine)
* [source](https://github.com/neoseele/continuous-deployment-on-kubernetes)

```sh
# create util cluster
gcloud beta container clusters create kube-util \
  --zone=us-central1-a \
  --num-nodes=2 \
  --min-nodes=2 \
  --max-nodes=4 \
  --image-type=COS \
  --enable-autoscaling \
  --enable-autorepair \
  --enable-autoupgrade \
  --enable-legacy-authorization \
  --scopes cloud-platform,storage-rw,logging-write,monitoring-write,service-control,service-management

# create ns
kubectl create namespace jenkins

# create pw and add it to jenkins/k8s/options
PASSWORD=`openssl rand -base64 15`; echo "Your password is $PASSWORD"; sed -i ".bak" "s#CHANGE_ME#$PASSWORD#" jenkins/k8s/options

# create secret
kubectl create secret generic jenkins --from-file=jenkins/k8s/options --namespace=jenkins
secret "jenkins" created

# create deployment
kubectl apply -f jenkins/k8s/jenkins.yaml
deployment "jenkins" created

# create services
kubectl apply -f jenkins/k8s/service_jenkins.yaml
service "jenkins-ui" created
service "jenkins-discovery" created

# create ssl key/cert
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/tls.key -out /tmp/tls.crt -subj "/CN=jenkins/O=jenkins"

# upload the key/cert to kube
kubectl create secret generic tls --from-file=/tmp/tls.crt --from-file=/tmp/tls.key --namespace jenkins

# create the load balancer
kubectl apply -f jenkins/k8s/lb/ingress.yaml

# create a node pool for canary
gcloud beta container node-pools create canary-pool \
  --zone=us-central1-a \
  --cluster=kube-util \
  --num-nodes=2 \
  --image-type=COS \
  --enable-autorepair \
  --enable-autoupgrade \
  --tags=kube-util-canary \
  --node-labels=env=canary \
  --scopes cloud-platform,storage-rw,logging-write,monitoring-write,service-control,service-management

# create a node pool for prod
gcloud beta container node-pools create prod-pool \
  --zone=us-central1-a \
  --cluster=kube-util \
  --num-nodes=2 \
  --image-type=COS \
  --enable-autorepair \
  --enable-autoupgrade \
  --tags=kube-util-prod \
  --node-labels=env=prod \
  --scopes cloud-platform,storage-rw,logging-write,monitoring-write,service-control,service-management
```
