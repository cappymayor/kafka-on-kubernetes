# HOW TO RUN THE SETUP

## Deployment with Strimzi & Helm

This section outlines how to spin up the Kafka infrastructure using Strimzi and Helm on a local Kubernetes cluster.

### prerequisites

- Local `Kubernetes cluster` like Minikube
- Install Kubectl
- Install Helm
- Create a repository on Dockerhub
- Create an s3 bucket on AWS called `kafka-on-kubernetes`
- Create a user and grant write access to the bucket ( Will be used by the connector )
  - Create an Access and Secret Key for that user.



Clone the repository.
```bash
git clone https://github.com/cappymayor/kafka-on-kubernetes.git
```

Start Minikube with increased resource allocations from the CLI.
```bash
minikube start --cpus 4 --memory 8000
```

Create a dedicated kubernetes namespace.
```bash
kubectl create ns strimzi-kafka
```

Install the strimzi operator chart dependency.
```bash
helm dependency build
```

Install the strimzi operator chart. Make sure the command is ran in the directory where the `Chart.yaml` is located.
```bash
helm install strimzi-operator . --namespace strimzi-kafka
```
After the strimzi operator is in `running` and `ready` state, next is to deploy Kafka cluster and the Kafka connector stacks.

Install the strimzi operator chart dependency.
```bash
kubectl apply -f kafka-cluster.yaml
```

Lets create all secrets that will be used by our stacks.
```bash
kubectl create secret generic aws-s3-credentials \
  --from-literal=AWS_ACCESS_KEY_ID="your_aws_access_key" \
  --from-literal=AWS_SECRET_ACCESS_KEY="your_aws_secret_access_key" \
  --namespace strimzi-kafka
```
```bash
 kubectl create secret docker-registry my-docker-credentials \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=your_dockerhub_username \
  --docker-password=your_dockerhub_personal_access_token \
  --docker-email=your_dockerhub_email \
  --namespace strimzi-kafka
```
```bash
kubectl create secret generic postgres-credentials \
  --namespace=strimzi-kafka \
  --from-literal=username='myusername' \
  --from-literal=password='mypassword'
  --namespace strimzi-kafka
```
