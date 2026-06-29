## Deployment with Strimzi & Helm

This section outlines how to spin up the Kafka infrastructure using Strimzi and Helm on a local Kubernetes cluster.

### prerequisites

- Local `Kubernetes cluster` like Minikube installed.
- Install Kubectl
- Install Helm
- Create a repository on Dockerhub `my-connect-cluster-debezium`
  - Be sure to [change this line](https://github.com/cappymayor/kafka-on-kubernetes/blob/master/postgres-debezium.yaml#L101) to match your dockerhub username after cloning.
- Create an s3 bucket on AWS called `kafka-on-kubernetes`
- Create a user and grant write access to the bucket ( Will be used by the connector )
  - Create an Access and Secret Key for that user.



### 1. Clone the repository.
```bash
git clone https://github.com/cappymayor/kafka-on-kubernetes.git
```

### 2. Start Minikube with increased resource allocations from the CLI.
```bash
minikube start --cpus 4 --memory 8000
```

### 3. Create a dedicated kubernetes namespace.
```bash
kubectl create ns strimzi-kafka
```

### 4. Install the strimzi operator chart dependency.
```bash
helm dependency build
```

### 5. Install the strimzi operator chart. Make sure the command is ran in the directory where the `Chart.yaml` is located.
```bash
helm install strimzi-operator . --namespace strimzi-kafka
```
### 6. After the strimzi operator is in `running` and `ready` state, next is to deploy Kafka cluster and the Kafka connector stacks.

### 7. Install the strimzi operator chart dependency.
```bash
kubectl apply -f kafka-cluster.yaml
```

### 8. Lets create all secrets that will be used by our stacks.
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

### 9. Deploy Postgres DB, Connect Cluster and Debezium Source Connector.
```bash
kubectl apply -f postgres-debezium.yaml
```

### 10. Deploy the s3 sink connector
```bash
kubectl apply -f s3-sink-connector.yaml
```

## Test the full pipeline

### 1. Create a table in the postgres database
```bash
kubectl exec -it deployment/postgres-db -n strimzi-kafka -- \
  psql -U myusername -d inventory -c \
  "CREATE TABLE public.customers (
     id SERIAL PRIMARY KEY,
     first_name VARCHAR(255),
     last_name VARCHAR(255),
     email VARCHAR(255)
   );"
```

### 2. Write some test record to the table
```bash
kubectl exec -it deployment/postgres-db -n strimzi-kafka -- \
psql -U myusername -d inventory -c "
INSERT INTO public.customers (first_name, last_name, email) VALUES 
('Elena', 'Rostova', 'elena.rostova@example.com'),
('Marcus', 'Vance', 'mvance@example.com'),
('Aiko', 'Tanaka', 'aiko_t@example.com'),
('Mateo', 'Silva', 'msilva@example.com'),
('Chloe', 'Dupont', 'chloe.dupont@example.com');"
```

Verify the write in your s3 bucket
