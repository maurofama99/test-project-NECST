# Performance behavior of microservice-based cloud application

## Local Machine implementation

### 1. Running a local K8s cluster

The simplest way to run a cluster on your local machine is by using Microk8s.

Install Microk8s:

```bash
sudo snap install microk8s --classic
```

Join the group:

```bash
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
su - $USER
```

Turn on the dns service:

```bash
cd <path-of-repo>/socialNetwork/wrk2
microk8s enable dns
```

### 2. Deploy the benchmark

Before you start get the IP address for the DNS resolver with dnsutils:

```bash
microk8s kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
```

```bash
microk8s kubectl exec -i -t dnsutils -- nslookup kube-dns.kube-system.svc.cluster.local
```

Then set it in files such as:
- `<path-of-repo>/socialNetwork/openshift/nginx-web-server-config/nginx.conf`
- `<path-of-repo>/socialNetwork/openshift/media-frontend-config/nginx.conf`

### 3. Deploy services

ATTENTION: to run the following script with microk8s, you need to add "microk8s " before every kubectl command in all the scripts that generates config maps and deploy services.

Run the script `<path-of-repo>/socialNetwork/openshift/scripts/deploy-all-services-and-configurations.sh`

### 4. Register users and construct social graphs

- Use `microk8s kubectl -n social-network get svc nginx-thrift` to get the Cluster-IP:NodePort.
- Paste it at `<path-of-repo>/socialNetwork/scripts/init_social_graph.py:72`

- Register users and construct social graph by running `cd <path-of-repo>/socialNetwork && python3 scripts/init_social_graph.py`.

## Cloud platform implementation

### Pre-requirements

- AWS CLI installed
- kOps installed

### 1. Log in into your AWS Management console

### 2. Configure your local machine with

```bash
aws configure
```
and set the region where you want to run your cluster.

### 3. Create an IAM role

You can create the kOps IAM user from the command line using the following:
```bash
aws iam create-group --group-name kops

aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops

aws iam create-user --user-name kops

aws iam add-user-to-group --user-name kops --group-name kops

aws iam create-access-key --user-name kops
```
or you can create it using the AWS Management Console.

### 4. Configure a cluster using Gossip

In order to use gossip-based DNS, configure the cluster domain name to end with `.k8s.local`

### 5. Cluster State storage

We need to create a dedicated S3 bucket for kops to use. We recommend keeping the creation of this bucket confined to YOUR REGION, otherwise more work will be required.

```bash
aws s3api create-bucket \
    --bucket prefix-example-com-state-store \
    --region us-east-1
```
Note: We STRONGLY recommend versioning your S3 bucket in case you ever need to revert or recover a previous state store
```bash
aws s3api put-bucket-versioning --bucket prefix-example-com-state-store  --versioning-configuration Status=Enabled
```

### 6. Prepare your local Enviroment

Let's set up a few environment variables to make the process easier.

```bash
export NAME=myfirstcluster.k8s.local
export KOPS_STATE_STORE=s3://prefix-example-com-state-store 
```
### 7. Create ssh key pair

This keypair is used for ssh into kubernetes cluster
```bash
ssh-keygen
```

### 8. Create your cluster

First create a configuration:
```bash
kops create cluster \
--state=${KOPS_STATE_STORE} \
--node-count=2 \
--master-size=t2.micro \
--node-size=t2.medium \
--zones=<YOUR REGION> \
--name=${KOPS_CLUSTER_NAME} \
--master-count 1
```

When you have your right configuration, finally build the cluster:
```bash
kops update cluster ${NAME} --yes
```

This will take a while, use the following commando to know when your cluster is ready:

```bash
kops validate cluster
```
### 9. Deploy the benchmark

Before you start get the IP address for the DNS resolver with dnsutils:

```bash
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
```

```bash
kubectl exec -i -t dnsutils -- nslookup kube-dns.kube-system.svc.cluster.local
```

Then set it in files such as:
- `<path-of-repo>/socialNetwork/openshift/nginx-web-server-config/nginx.conf`
- `<path-of-repo>/socialNetwork/openshift/media-frontend-config/nginx.conf`

#### Deploy services

Run the script `<path-of-repo>/socialNetwork/openshift/scripts/deploy-all-services-and-configurations.sh`

### 10. Use an SSH tunnel to expose nginx-thrift service

 - Use `kubectl -n social-network get svc nginx-thrift` to get the nginx-thrift NodePort.

Start the SSH tunnel using Session Manager:
```bash
ssh -i /path/my-key-pair.pem username@instance-id -L <NodePort>:targethost:<NodePort>
```
### 11. Register users and construct social graphs

- Register users and construct social graph by running `cd <path-of-repo>/socialNetwork && python3 scripts/init_social_graph.py`.

### 12. Delete your cluster

Running a Kubernetes cluster within AWS obviously costs money, and so you may want to delete your cluster if you are finished running experiments.

```bash
kops delete cluster --name ${NAME} --state=${KOPS_STATE_STORE} --yes
```

## Running HTTP workload generator

First, make sure that the `wrk` command has been properly built for your platform:
```bash
cd <path-of-repo>/socialNetwork/wrk2
make clean
make
```
For all load generating commands below, use the NodePort of nginx-thrift service.

### Read home timelines

```bash
cd <path-of-repo>/socialNetwork/wrk2
./wrk -t2 -c100 -d30s --u_latency -s ./scripts/social-network/read-home-timeline.lua http://localhost:<NodePort>/wrk2-api/home-timeline/read -R 2000
```

### Read user timelines

```bash
cd <path-of-repo>/socialNetwork/wrk2
./wrk -t2 -c100 -d30s --u_latency -s ./scripts/social-network/read-user-timeline.lua http://localhost:<NodePort>/wrk2-api/user-timeline/read -R2000
```

Don't use compose posts script, it doesn't work beacuse of a unresolved issue.

## Metrics-server

To run metrics-server, deploy the `components.yaml` file in the `kube-system` namespace of your cluster, use `kubectl top pods` to start monitoring pods.
