# Confluent Platform Deploymemt
## Requirements
### CLI Tools
* kubectl cli v1.34: https://dl.k8s.io/release/v1.34.0/bin/linux/amd64/kubectl
* vcf cli v9.0.1: https://packages.broadcom.com/artifactory/vcf-distro/vcf-cli/linux/amd64/v9.0.1/
* helm cli v3.19: https://get.helm.sh/helm-v3.19.0-linux-amd64.tar.gz

### Get files
```shell
git clone https://github.com/cleeistaken/confluent-platform-k8s-testgin.git
```

### Required vSphere Steps
1. Create a name space names '**cp-ns**'
2. Add '**vsan-esa-default-policy-raid5**' storage policy to '**cp-ns**' namespace
3. Create and add a VM class named '**cp-vm-class**' and add it to '**cp-ns**' namespace
   * 
4. Add '**best-effort-medium**' VM class to '**cp-ns**' namespace

## Deployment Procedure

### 1. Set variables
```shell
# Update with correct values!
SUPERVISOR_IP="10.0.0.1"
SUPERVISOR_USERNAME="user@domain"
NAMESPACE_NAME="cp-ns"
CLUSTER_NAME="cp-vks"
TUTORIAL_HOME="https://raw.githubusercontent.com/confluentinc/confluent-kubernetes-examples/master/quickstart-deploy/kraft-quickstart"
```

### 2. Clean kubectl and vcf configs
```shell
rm ~/.kube/config
rm -rf ~/.config/vcf/
```

### 3. Login to supervisor
```shell
# Login and get contexts
kubectl vsphere login --server=$SUPERVISOR_IP  --insecure-skip-tls-verify=true -u $SUPERVISOR_USERNAME
kubectl config get-contexts
```

### 4. Create context on supervisor
```shell
# Create a context named 'cp'
vcf context create cp --endpoint $SUPERVISOR_IP --insecure-skip-tls-verify -u $SUPERVISOR_USERNAME 
vcf context use cp
```

### 5. Create VKS cluster
```shell
# Create a VKS cluster as defined in vks.yaml
kubectl apply -f cp-vks.yaml
```

### 6. Connect to VKS cluster
```shell
# Connect to the VKS cluster
vcf context create vks --endpoint $SUPERVISOR_IP --insecure-skip-tls-verify -u $SUPERVISOR_USERNAME --workload-cluster-namespace=$NAMESPACE_NAME --workload-cluster-name=$CLUSTER_NAME
vcf context use vks:$CLUSTER_NAME
```

### 8. Create confluent namespace
```shell
# Create a namespace on the VKS cluster
kubectl create namespace confluent
kubectl config set-context --current --namespace=confluent
```

### 9. Install Confluent Operator
```shell
### Add and install Confluent Operator
helm repo add confluentinc https://packages.confluent.io/helm
helm repo update
helm upgrade --install confluent-operator confluentinc/confluent-for-kubernetes

# Check to make sure the Confluent Operator Pod is 1/1 Ready and Running 
kubectl get pods
```

### 10. Deploy Confluent Platform Components
```shell
# Components
kubectl apply -f $TUTORIAL_HOME/confluent-platform-c3++.yaml

# Sample producer
kubectl apply -f $TUTORIAL_HOME/producer-app-data.yaml

# Check to make sure the componnets are 1/1 Ready and Running 
kubectl get pods
```

### 11. Expose ports
```shell
# Expose Control Center
kubectl expose service controlcenter-0-internal  --type=LoadBalancer --name=controlcenter-0-external
```


## Cleanup Procedure

```shell
# Shut down the sample producer app
kubectl delete -f $TUTORIAL_HOME/producer-app-data.yaml

#Shut down the Confluent Platform clusters:
kubectl delete -f $TUTORIAL_HOME/confluent-platform.yaml

# Shut down the CFK
helm uninstall confluent-operator

#Delete the namespace
kubectl delete namespace confluent
```


## Troubleshooting

### Useful Commands
```shell
kubectl get all

kubectl get pods -o wide

kubectl logs -f <container name>
```

### Login to Docker.io 
May be required if the deployment hits errors about the site hitting image pull limits.

#### Create Secret with Docker.io Credentials
```shell
kubectl create secret docker-registry docker-io-secret \
  --docker-server=docker.io \
  --docker-username=<docker username> \
  --docker-password=<docker password> \
  --docker-email=<docker email> 
```

#### Modify confluent-platform-c3++.yaml
##### Before
```yaml
apiVersion: platform.confluent.io/v1beta1
kind: KRaftController
metadata:
  name: kraftcontroller
  namespace: confluent
spec:
  dataVolumeCapacity: 10G
  image:
    application: docker.io/confluentinc/cp-server:8.1.0
    init: confluentinc/confluent-init-container:3.1.0
  replicas: 3
  dependencies:
    metricsClient:
      url: http://controlcenter.confluent.svc.cluster.local:9090
```

##### After
```yaml
apiVersion: platform.confluent.io/v1beta1
kind: KRaftController
metadata:
  name: kraftcontroller
  namespace: confluent
spec:
  dataVolumeCapacity: 10G
  image:
    application: docker.io/confluentinc/cp-server:8.1.0
    init: confluentinc/confluent-init-container:3.1.0
    pullSecretRef:           # <----
      - docker-io-secret     # <----
  replicas: 3
  dependencies:
    metricsClient:
      url: http://controlcenter.confluent.svc.cluster.local:9090
```

