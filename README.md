# HOW TO RUN THE SETUP

## Deployment with Strimzi & Helm

This section outlines how to spin up the Kafka infrastructure using Strimzi and Helm on a local Kubernetes cluster.

### Installation prerequisites

- Local `Kubernetes cluster` like Minikube
- Kubectl
- Helm


- Clone the repository.
```bash
git clone https://github.com/cappymayor/kafka-on-kubernetes.git
```

- Start Minikube with increased resource allocations from the CLI.
```bash
minikube start --cpus 4 --memory 8000
```

- Create a dedicated kubernetes namespace.
```bash
kubectl create ns strimzi-kafka
```

- Install the strimzi operator chart dependency.
```bash
helm dependency build
```

- Install the strimzi operator chart. Make sure the command is ran in the directory where the `Chart.yaml` is located.
```bash
helm install strimzi-operator . --namespace strimzi-kafka
```
After the strimzi operator is in `running` and `ready` state, next is to deploy Kafka cluster and the Kafka connector stacks.

- Install the strimzi operator chart dependency.
```bash
kubectl apply -f kafka-cluster.yaml
```
