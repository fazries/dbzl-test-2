# dbzl-test-2
## Building EKS environment
### create a control plane and worker node
```
eksctl create cluster \
--name sre-setyadi \
--version 1.14 \
--nodegroup-name standard-workers \
--node-type t3.medium \
--nodes 3 \
--nodes-min 1 \
--nodes-max 4 \
--node-ami auto

eksctl create nodegroup \
--cluster sre-setyadi \
--version auto \
--name dbzl-workers-1 \
--node-type m5.large \
--node-ami auto \
--nodes 3 \
--nodes-min 1 \
--nodes-max 6
```
check the cluster condition
```
aws eks --region eu-west-1 describe-cluster --name sre-setyadi --query cluster.status
```
update kubeconfig
```
aws eks --region eu-west-1 update-kubeconfig --name sre-setyadi
```

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
there's a chart named webapp. you can update the parameters via `values.yaml`

### running the deployment
```
helm install webapp -n webapp
```
it will create an kubernetes object:
- apps pod
- deployment
- services
- configmaps
- hpa
- ingress [optional]

or you can try to install using manifest file at `dbzl-test-2/k8s/manifest/application`
service of the application can be access via
### http://af547bafce12811e9b8570a47d626478-1348575436.eu-west-1.elb.amazonaws.com


## log management
using Elasticsearch, Fluentd and Kibana
installation:
```
cd dbzl-test-2/k8s/manifest/log
kubectl apply -f 01.elasticsearch-statefullset.yaml  02.fluentd-rbac.yaml  03.fluentd.yaml  04.kibana.yaml
```
