# dbzl-test-2
## Building EKS environment
### create a control plane and worker node
```
eksctl create cluster \
--name cluster-sre-setyadi \
--version 1.14 \
--nodegroup-name standard-workers \
--node-type t3.medium \
--nodes 3 \
--nodes-min 1 \
--nodes-max 4 \
--node-ami auto

eksctl create nodegroup \
--cluster cluster-sre-setyadi \
--version auto \
--name sre-setyadi-2 \
--node-type m5.large \
--node-ami auto \
--nodes 3 \
--nodes-min 1 \
--nodes-max 3
```
check the cluster condition
```
aws eks --region eu-west-1 describe-cluster --name cluster-sre-setyadi --query cluster.status
```
update kubeconfig
```
aws eks --region eu-west-1 update-kubeconfig --name cluster-sre-setyadi
```

it will creating a control plane and node worker


### ingress installation
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.2/docs/examples/iam-policy.json
 aws iam create-policy \
--policy-name ALBIngressControllerIAMPolicy \
--policy-document file://iam-policy.json
```
apply this policy into instance role of worker nodes

### helm installation
do init at your local
```
helm init
```
setup rbac for helm
```
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```
# Deployment
### containerize the application
this process only containerize the application code, and encapsulate it with php-fpm
```
cd dbzl-test-2
docker build -t [tagname:version] .
docker push [tagname:version]
```
nginx server and all deployment including configuration will be done by `helm` chart

## deploying using helm chart
```
cd dbzl-test-2
cd k8s/helm
```
there's a chart named `webapp`. you can update the parameters via `values.yaml`

### running the deployment
```
helm install webapp -n webapp
```
it will create an kubernetes object:
- pods webapps
- deployment
- services
- configmaps
- hpa
- ingress [optional]

or you can try to install using manifest file at `dbzl-test-2/k8s/manifest/application`
service of the application can be access via
### aa5790745e1ea11e9b6ec067b079aa35-508921207.eu-west-1.elb.amazonaws.com 

## log management
## deploying using helm cart
```
cd dbzl-test-2
cd k8s/helm
```
there's a chart named `elk`. you can update the parameters via `values.yaml`

### running the deployment
```
helm install elk -n elk
```
it will create a kubernetes object:
- statefulset pod of elasticsearch and kibana
- persistent volume
- services

service of kibana can be access via
### http://a35ff89d3e20611e9b6ec067b079aa35-2130241691.eu-west-1.elb.amazonaws.com/app/kibana#/home?_g=()

kibana search webapp
### http://a35ff89d3e20611e9b6ec067b079aa35-2130241691.eu-west-1.elb.amazonaws.com/app/kibana#/discover?_g=()&_a=(columns:!(_source),filters:!(('$state':(store:appState),meta:(alias:!n,disabled:!f,index:'40c86a10-e209-11e9-983d-537ffefdb259',key:kubernetes.pod_name,negate:!f,params:(query:webapp-,type:phrase),type:phrase,value:webapp-),query:(match:(kubernetes.pod_name:(query:webapp-,type:phrase))))),index:'40c86a10-e209-11e9-983d-537ffefdb259',interval:auto,query:(language:lucene,query:''),sort:!('@timestamp',desc))

## manual installation using manifest file
using Elasticsearch, Fluentd and Kibana
installation:
```
cd dbzl-test-2/k8s/manifest/elk
kubectl apply -f 01.elasticsearch-statefullset.yaml  -f 02.fluentd.yaml -f 03.kibana.yaml 
```
